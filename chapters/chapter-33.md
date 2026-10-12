# Глава 33. Notifications API

Уведомления — единственный механизм Web Platform, который позволяет приложению обратиться к пользователю **вне контекста вкладки**. Это одновременно и самая мощная, и самая обманчивая возможность браузера. Мощная — потому что уведомления переживают закрытие страницы и работают как канал доставки от Service Worker. Обманчивая — потому что API рапортует об успехе задолго до того, как пользователь действительно что-то увидит.

В этой главе мы разберём три уровня Notifications API, спроектируем полноценный диагностический инструментарий и разберём ловушку, в которую попадает каждый, кто впервые работает с уведомлениями: **браузер может успешно показать уведомление, а операционная система — молча его отклонить**.

## 33.1. Три уровня Notifications API

Notifications API существует в трёх взаимосвязанных слоях, и путать их — источник большинства проблем.

```text
┌─────────────────────────────────────────────────────────────────┐
│  Уровень 1: PERMISSION                                          │
│  ─────────────────────                                          │
│  Notification.permission: 'default' | 'granted' | 'denied'      │
│  Sticky per origin. Один раз denied — диалог больше не вызвать. │
├─────────────────────────────────────────────────────────────────┤
│  Уровень 2: PAGE NOTIFICATION                                   │
│  ────────────────────────────                                   │
│  new Notification(title, options)                               │
│  Живёт, пока открыта вкладка. События show/click/close приходят │
│  в страницу через notification.onshow / .onclick / .onclose.    │
├─────────────────────────────────────────────────────────────────┤
│  Уровень 3: SERVICE WORKER NOTIFICATION                         │
│  ──────────────────────────────────────                         │
│  registration.showNotification(title, options)                  │
│  Живёт вне вкладки. Закрыли страницу — попап остался.           │
│  Клик приходит в SW через 'notificationclick'.                  │
│  SW отправляет сообщение обратно через clients.postMessage.     │
└─────────────────────────────────────────────────────────────────┘
```

Второй и третий уровни кажутся похожими, но это принципиально разные механизмы:

| Свойство | Page notification | SW notification |
|---|---|---|
| Живёт после закрытия вкладки | Нет | Да |
| Клик приходит в | `Notification.onclick` страницы | событие `notificationclick` в SW |
| Контекст исполнения | Renderer Process (вкладка) | Service Worker (отдельный поток) |
| События доступны через | `NotificationEventTarget` | `ExtendableEvent` в SW |
| Основа для push-уведомлений | Нет | Да |

Страничные уведомления — это быстрый способ привлечь внимание, пока пользователь на сайте. SW-уведомления — фундамент для push-сценариев, когда сервер шлёт сигнал на устройство, а SW показывает попап даже при закрытой вкладке.

## 33.2. Permission: единственная стена

Разрешение — единственный барьер между вашим кодом и диалогом ОС. И это стена, которую нельзя перепрыгнуть программно.

```rust
pub enum Permission {
    Default,   // ещё не спрашивали
    Granted,   // разрешено
    Denied,    // отказано — sticky per origin
    Unknown,   // API недоступен или значение не распознано
}
```

Читаем состояние через статическое свойство конструктора `Notification.permission`:

```rust
pub fn permission() -> Permission {
    let ctor = notification_ctor()?;
    let p = js_sys::Reflect::get(&ctor, &JsValue::from_str("permission"))?;
    let s = p.as_string().unwrap_or_else(|| "unknown".to_string());
    Permission::from_str(&s)
}
```

Запрашиваем только из user gesture (клик по кнопке):

```rust
pub async fn request_permission() -> Result<Permission, JsValue> {
    let ctor = notification_ctor()?;
    let f: js_sys::Function = Reflect::get(&ctor, "requestPermission")?;
    let result = f.call0(&ctor)?;

    // Современные браузеры возвращают Promise, старые — callback.
    if let Some(p) = result.dyn_ref::<js_sys::Promise>() {
        let awaited = JsFuture::from(p.clone()).await?;
        let s = awaited.as_string().unwrap_or_else(|| "unknown".into());
        return Ok(Permission::from_str(&s));
    }
    if let Some(s) = result.as_string() {
        return Ok(Permission::from_str(&s));
    }
    Ok(Permission::Unknown)
}
```

Три важных правила работы с permission:

**Правило 1. `denied` — навсегда.** После первого отказа JS не может повторно вызвать диалог. Пользователь должен зайти в настройки сайта вручную. Поэтому запрашивайте разрешение только тогда, когда у вас есть что показать — не при загрузке страницы, а при первом осмысленном действии.

**Правило 2. Только из user gesture.** В Firefox без клика `requestPermission()` бросает `SecurityError`. В Chrome без клика промис молча никогда не разрешится — зависнете навсегда.

**Правило 3. Перечитывайте при focus.** Пользователь мог изменить настройку в адресной строке или системных настройках, пока вкладка была в фоне. Слушаем `window.focus` и обновляем badge.

## 33.3. Page notification

Конструктор `new Notification(title, options)` создаёт попап, привязанный к вкладке. На Rust/web-sys работаем через `Reflect::construct`, потому что фича `Notification` не подключена в `Cargo.toml`:

```rust
let args = js_sys::Array::new();
args.push(&JsValue::from_str(&opts.title));
args.push(&build_notify_options(&opts));

let notification = js_sys::Reflect::construct(
    &ctor.dyn_into::<js_sys::Function>()?,
    &args,
)?;
```

События жизненного цикла навешиваются как свойства — `onshow`, `onclick`, `onclose`, `onerror`:

```rust
for (event_name, label) in [
    ("show", "show"),
    ("click", "click"),
    ("close", "close"),
    ("error", "error"),
] {
    let closure = Closure::<dyn FnMut(web_sys::Event)>::new(
        move |_e| { on_event(label); },
    );
    Reflect::set(
        &notification,
        &JsValue::from_str(&format!("on{event_name}")),
        closure.as_ref().unchecked_ref(),
    )?;
    closures.push(closure);
}
```

`Closure` нужно удерживать в `NotificationHandle`, иначе он соберётся GC и обработчики перестанут вызываться.

## 33.4. Service Worker notification

Ключевое отличие SW-уведомления — оно не привязано к вкладке. Регистрируем SW и вызываем `showNotification` на его registration:

```rust
pub async fn show_via_registration(
    reg: &JsValue,
    opts: &NotifyOptions,
) -> Result<(), JsValue> {
    let show_fn: js_sys::Function =
        Reflect::get(reg, &JsValue::from_str("showNotification"))?.dyn_into()?;
    let promise = show_fn.call2(
        reg,
        &JsValue::from_str(&opts.title),
        &build_options_js(opts),
    )?;
    JsFuture::from(promise.dyn_into::<js_sys::Promise>()?).await?;
    Ok(())
}
```

Клик по такому попапу приходит **не в страницу**, а в Service Worker — событие `notificationclick`. SW должен сам решить, что делать: сфокусировать существующую вкладку, открыть новую или отправить сообщение всем открытым клиентам.

В `sw.js`:

```javascript
self.addEventListener('notificationclick', (event) => {
    event.notification.close();
    const tag = event.notification.tag || '';

    event.waitUntil((async () => {
        const clients = await self.clients.matchAll({
            type: 'window',
            includeUncontrolled: true,
        });

        // Оповещаем страницы о клике.
        for (const client of clients) {
            client.postMessage({ type: 'notification-click', tag });
        }

        // Фокусируем первую вкладку или открываем новую.
        for (const client of clients) {
            if ('focus' in client) return client.focus();
        }
        if (self.clients.openWindow) {
            return self.clients.openWindow('/');
        }
    })());
});
```

На странице слушаем `navigator.serviceWorker.addEventListener('message', ...)`:

```rust
let sw_val = js_sys::Reflect::get(&nav, &JsValue::from_str("serviceWorker"))?;
let target = sw_val.dyn_into::<web_sys::EventTarget>()?;
target.add_event_listener_with_callback(
    "message",
    cb.as_ref().unchecked_ref(),
)?;
```

Так замыкается цикл: SW получил клик → отправил сообщение во все вкладки → страница получила `notification-click` и обновила UI. Это отличие от page-уведомлений, где клик приходит напрямую в `Notification.onclick` страницы.

## 33.5. Инсайт: браузер и ОС — два разных судьи

А теперь самое важное. Всё, что мы написали выше, — корректно. И всё же уведомление может **не появиться**.

### Что происходит на самом деле

Когда вы вызываете `showNotification()` или `new Notification()`, происходит следующее:

```text
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Rust / WASM   │ →  │  Браузер (FF/  │ →  │  Операционная  │
│  ваш код       │    │  Chrome)       │    │  система       │
└────────────────┘    └────────────────┘    └────────────────┘
       │                     │                     │
       │  new Notification() │                     │
       │────────────────────▶│                     │
       │                     │  IPC → Network      │
       │                     │  Service → system   │
       │                     │  notification API   │
       │                     │────────────────────▶│
       │                     │                     │
       │  Promise resolved   │                     │
       │  ◀──────────────────│                     │
       │                     │                     │
       │  событие 'show'     │                     │
       │  ◀──────────────────│                     │
       │                     │                     │
       │  getNotifications() │                     │
       │────────────────────▶│                     │
       │  []  ← пусто!       │                     │
       │  ◀──────────────────│                     │
                             │                     │
                             │     ОС решила:      │
                             │  "не показывать"    │
```

**Браузер рапортует об успехе раньше, чем ОС приняла окончательное решение.** Событие `show` означает лишь, что браузер передал уведомление дальше по цепочке. Промис `showNotification()` разрешается, когда браузер закончил свою часть работы. А ОС может молча отклонить попап — и вы об этом узнаете только по косвенным признакам.

### Надёжный индикатор: `getNotifications()`

Единственный способ проверить, приняла ли ОС уведомление, — запросить список активных попапов через `registration.getNotifications()`:

```rust
pub async fn count_via_registration(reg: &JsValue) -> Result<u32, JsValue> {
    let f: js_sys::Function =
        Reflect::get(reg, &JsValue::from_str("getNotifications"))?.dyn_into()?;
    let promise = f.call0(reg)?;
    let result: JsValue = JsFuture::from(
        promise.dyn_into::<js_sys::Promise>()?,
    ).await?;
    let arr: js_sys::Array = result.dyn_into()?;
    Ok(arr.length())
}
```

Если `showNotification()` разрешился, а `getNotifications().length = 0` — это **не баг кода**. Это отказ ОС.

### Шесть причин отказа ОС

Проверено на практике — в порядке убывания вероятности:

1. **Режим «Не беспокоить» / Focus Assist.** Самая частая причина. В Windows: Параметры → Система → Уведомления → «Не беспокоить». В GNOME: Параметры → Уведомления → режим «Не беспокоить».
2. **Уведомления для приложения отключены в системных настройках.** В Windows 11 глобальный переключатель может быть включён, но для Firefox/Chrome конкретно — выключен. Прокрутите список приложений вниз и проверьте.
3. **Галочка «Показывать баннеры уведомлений».** Уведомление может попадать в Центр уведомлений, но не показываться как всплывающий баннер. Проверьте настройки приложения в системных параметрах.
4. **Окно браузера не в фокусе.** Windows и некоторые Linux-окружения не показывают баннеры, пока окно браузера неактивно. Проверяется через `document.hasFocus()`.
5. **Активное полноэкранное приложение.** Другое приложение в fullscreen (игра, презентация, видеоплеер) подавляет все баннеры системы.
6. **WSL / VM / удалённый сеанс.** Если ваш сервер разработки запущен внутри WSL, VM или SSH-сессии, GUI-уведомления могут не пробрасываться на хост. Проверьте командой `notify-send "тест"` в терминале WSL.

### Что говорит браузер и что говорит ОС

Полезно различать три разных «успеха»:

| Что вы видите | Что это значит |
|---|---|
| `new Notification()` не бросил исключение | Браузер принял вызов, синтаксис корректен |
| Промис `showNotification()` разрешился | Браузер передал уведомление в ОС |
| Событие `show` пришло | Браузер получил подтверждение от ОС о приёме |
| `getNotifications().length > 0` | **ОС действительно зарегистрировала попап** |
| Пользователь видит баннер | Всё выше + ОС решила показать, а не только принять |

Только последние две строки означают, что уведомление реально работает. Всё, что выше, — промежуточные состояния.

## 33.6. Диагностический инструментарий

Раз API так легко обманывает, диагностика становится обязательной частью любого демо. Собираем снимок окружения, который сразу показывает, где теряется уведомление:

```rust
pub async fn collect_diagnostics() -> Vec<(String, String)> {
    let mut out: Vec<(String, String)> = Vec::new();
    let global = js_sys::global();

    // Secure context — фундаментальное требование.
    out.push(("isSecureContext".into(),
        yes_no(js_get_bool(&global, "isSecureContext").unwrap_or(false))));

    // Notification API доступен?
    out.push(("Notification API".into(), yes_no(is_supported())));

    // Legacy permission.
    out.push(("Notification.permission (legacy)".into(),
        permission().as_str().into()));

    // Permissions API — более надёжный источник.
    out.push(("Permissions API (notifications)".into(),
        permission_via_permissions_api().await
            .unwrap_or_else(|| "n/a".into())));

    // maxActions = 2 в Chrome, undefined в старых движках.
    let ma = notification_ctor().ok()
        .and_then(|c| js_get(&c, "maxActions").and_then(|v| v.as_f64()));
    out.push(("Notification.maxActions".into(),
        ma.map(|n| n.to_string()).unwrap_or_else(|| "n/a".into())));

    // navigator: UA, platform, webdriver, headless.
    if let Some(nav) = js_get(&global, "navigator") {
        let ua = js_get_str(&nav, "userAgent").unwrap_or_default();
        out.push(("navigator.userAgent".into(), ua.clone()));
        out.push(("navigator.platform".into(),
            js_get_str(&nav, "platform").unwrap_or_default()));
        out.push(("navigator.webdriver".into(),
            yes_no(js_get_bool(&nav, "webdriver").unwrap_or(false))));
        let ua_lower = ua.to_lowercase();
        out.push(("UA содержит 'headless'".into(),
            yes_no(ua_lower.contains("headless"))));
    }

    // document: visibilityState, hasFocus — критично для Windows.
    if let Some(doc) = js_get(&global, "document") {
        out.push(("document.visibilityState".into(),
            js_get_str(&doc, "visibilityState").unwrap_or_default()));
        out.push(("document.hasFocus()".into(),
            yes_no(is_window_focused())));
    }

    // Service Worker: состояние регистрации и getNotifications().
    out.push(("Service Worker API".into(),
        yes_no(crate::notif_sw::is_supported())));

    match crate::notif_sw::get_registration().await {
        Ok(Some(reg)) => {
            out.push(("SW registration".into(), "present".into()));
            for key in ["active", "installing", "waiting"] {
                if let Some(v) = js_get(&reg, key) {
                    if !v.is_null() && !v.is_undefined() {
                        let state = js_get_str(&v, "state").unwrap_or_default();
                        out.push((format!("SW.{key}.state"), state));
                    }
                }
            }
            match crate::notif_sw::count_via_registration(&reg).await {
                Ok(n) => out.push((
                    "SW getNotifications().length".into(),
                    n.to_string(),
                )),
                Err(e) => out.push((
                    "SW getNotifications()".into(),
                    format!("error: {:?}", e),
                )),
            }
        }
        Ok(None) => out.push((
            "SW registration".into(),
            "none — нажмите «Регистрировать SW»".into(),
        )),
        Err(e) => out.push((
            "SW registration".into(),
            format!("error: {:?}", e),
        )),
    }

    // Подсказки.
    let mut hints: Vec<&'static str> = Vec::new();
    if !js_get_bool(&global, "isSecureContext").unwrap_or(false) {
        hints.push("Нет secure context — Notification API не работает.");
    }
    if !is_supported() {
        hints.push("Notification API недоступен в этом браузере/режиме.");
    }
    if permission() != Permission::Granted {
        hints.push("Разрешение не granted — попап не покажется.");
    }
    if !is_window_focused() {
        hints.push(
            "document.hasFocus() = false. Windows и некоторые Linux-\
             окружения НЕ показывают баннеры, пока окно браузера не в \
             фокусе. Кликните по окну браузера и повторите.",
        );
    }
    if let Some(nav) = js_get(&global, "navigator") {
        if js_get_bool(&nav, "webdriver").unwrap_or(false) {
            hints.push(
                "navigator.webdriver = true (автотест). ОС часто глушит \
                 уведомления в этом режиме.",
            );
        }
        let ua = js_get_str(&nav, "userAgent").unwrap_or_default().to_lowercase();
        if ua.contains("headless") {
            hints.push("Headless-браузер — попап может не отображаться ОС.");
        }
    }
    hints.push(
        "Если событие show пришло, но попап не виден — это ОС, а не \
         браузер. Порядок проверки: (1) режим «Не беспокоить» / Focus; \
         (2) уведомления для Firefox/Chrome включены в системных \
         настройках; (3) галочка «Показывать баннеры уведомлений»; \
         (4) фокус окна браузера; (5) активное полноэкранное \
         приложение; (6) проброс GUI в WSL/VM (notify-send).",
    );

    out.push(("— подсказки —".into(), String::new()));
    for h in hints {
        out.push(("•".into(), h.to_string()));
    }
    out
}
```

Дополнительно проверяем фокус окна:

```rust
pub fn is_window_focused() -> bool {
    let global = js_sys::global();
    let doc = match js_get(&global, "document") {
        Some(d) => d,
        None => return false,
    };
    js_get(&doc, "hasFocus")
        .and_then(|f| f.dyn_into::<js_sys::Function>().ok())
        .and_then(|f| f.call0(&doc).ok())
        .and_then(|v| v.as_bool())
        .unwrap_or(false)
}
```

## 33.7. Полный цикл показа уведомления

Собираем всё вместе — от клика по кнопке до подтверждения от ОС:

```rust
fn show_page_notification(
    app: Rc<App>,
    title: String,
    body: String,
    tag: String,
    require_interaction: bool,
) {
    // 1. Проверяем permission.
    if notif_core::permission() != Permission::Granted {
        set_status(&app.ui.page_status, "err",
            "сначала разрешите уведомления");
        return;
    }

    // 2. Предупреждаем о фокусе ДО показа — иначе пользователь
    //    не поймёт, почему попап не виден.
    if !notif_core::is_window_focused() {
        app.push_log(
            LogKind::Warn,
            "окно не в фокусе — Windows может не показать баннер. \
             Кликните по окну браузера перед показом.".to_string(),
        );
    }

    // 3. Создаём уведомление.
    let opts = NotifyOptions {
        title: title.clone(),
        body,
        tag,
        icon: None,
        require_interaction,
    };

    let saw_show = Rc::new(Cell::new(false));
    let saw_event = Rc::new(Cell::new(false));

    let on_event = move |event_name: &str| {
        saw_event.set(true);
        match event_name {
            "show" => {
                saw_show.set(true);
                // НЕ рапортуем «показан» — только «браузер передал в ОС».
                set_status(
                    &status_el,
                    "busy",
                    "браузер передал в ОС — ждём визуальной проверки…",
                );
            }
            "error" => set_status(
                &status_el,
                "err",
                "браузер сообщил об ошибке показа",
            ),
            _ => set_status(&status_el, "ok", &msg),
        }
    };

    // 4. Показываем.
    match app.page_manager.show(opts, on_event) {
        Ok(()) => {
            // 5. Safety-net: если через 2 секунды ни одного события
            //    не пришло — что-то фундаментальное сломалось.
            Timeout::new(SHOW_TIMEOUT_MS, move || {
                if !saw_event.get() {
                    app_t.push_log(LogKind::Warn,
                        "за 2с не пришло ни одного события");
                } else if saw_show.get() {
                    app_t.push_log(LogKind::Info,
                        "событие show получено — если попап не виден, \
                         это отказ ОС (см. «Диагностику»)".to_string());
                    set_status(
                        &app_t.ui.page_status,
                        "ok",
                        "браузер передал в ОС. Если попап не виден — \
                         отказ ОС. См. «Диагностику».",
                    );
                }
            }).forget();
        }
        Err(e) => {
            set_status(&app.ui.page_status, "err",
                &format!("show: {:?}", e));
        }
    }
}
```

Ключевое отличие от наивной реализации: **мы не считаем факт `show` гарантией видимого попапа**. Статус «показан» мы заменяем на «браузер передал в ОС, ждём визуальной проверки», а окончательный вердикт оставляем пользователю.

## 33.8. Чек-лист при отладке

Если попап не появляется, хотя все проверки прошли:

```text
┌─────────────────────────────────────────────────────────────────┐
│  ЧЕК-ЛИСТ ОТЛАДКИ УВЕДОМЛЕНИЙ                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ isSecureContext = true                                       │
│  ✓ Notification.permission = 'granted'                          │
│  ✓ Permissions API (notifications) = 'granted'                  │
│  ✓ Событие 'show' пришло в консоль                              │
│                                                                 │
│  Если getNotifications().length = 0 → отказ ОС:                 │
│                                                                 │
│  1. Windows: Параметры → Система → Уведомления                  │
│     • «Не беспокоить» = ВЫКЛ                                    │
│     • Firefox/Chrome в списке = ВКЛ                             │
│     • «Показывать баннеры уведомлений» = ВКЛ                    │
│                                                                 │
│  2. Linux (GNOME/KDE):                                          │
│     • Параметры → Уведомления → режим «Не беспокоить» = ВЫКЛ    │
│     • Разрешение для браузера = ВКЛ                             │
│                                                                 │
│  3. Фокус: кликните по окну браузера                            │
│     document.hasFocus() должен стать true                       │
│                                                                 │
│  4. Полноэкранные приложения: закройте другие fullscreen-окна   │
│                                                                 │
│  5. WSL/VM: проверьте `notify-send "тест"` в терминале          │
│     Если не работает — проблема в пробросе GUI                  │
│                                                                 │
│  6. Firefox: убедитесь, что вы взаимодействовали со страницей   │
│     (кликнули по ней) в текущей сессии — без этого Firefox      │
│     не показывает уведомления                                   │
│                                                                 │
│  7. about:config → alerts.useSystemBackend                      │
│     Попробуйте переключить значение и перезапустить браузер     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 33.9. Архитектурные выводы

Notifications API — редкий пример в Web Platform, когда **успех API не означает успех пользовательского сценария**. Из этого следуют три архитектурных правила.

**Правило 1. Разделяйте уровни успеха.** Не считайте `new Notification()` достаточным доказательством. Различайте: конструктор не упал → промис разрешился → событие `show` пришло → `getNotifications().length > 0` → пользователь видит баннер. Каждый следующий уровень строже предыдущего.

**Правило 2. Диагностика — часть демо, а не опция.** Если вы пишете демо уведомлений, обязательно встраивайте панель диагностики. Иначе вы будете ловить «баги» в коде, которых там нет.

**Правило 3. Сообщайте пользователю правду.** Статус «уведомление показано» — ложь в 30% случаев. Статус «браузер передал в ОС, если попап не виден — проверьте системные настройки» — честный и полезный.

## 33.10. Что мы построили

Полноценное демо Главы 33 включает:

1. **Модуль `notif_core`** — Permission, `NotificationManager`, `NotifyOptions`, сбор диагностики.
2. **Модуль `notif_sw`** — регистрация Service Worker, `showNotification` через registration, `getNotifications`, массовое закрытие.
3. **Файл `sw.js`** — обработчики `notificationclick` и `notificationclose` с `clients.postMessage`.
4. **Компонент `app`** — четыре панели UI (Permission, Page notification, SW notification, Журнал) плюс диагностическая панель на всю ширину.
5. **Диагностический инструментарий** — снимок окружения, проверка фокуса, чек-лист отладки.

Ключевой инсайт, вынесенный из главы: **браузер и операционная система — два разных судьи**. Браузер решает, разрешено ли вашему origin показывать уведомления. ОС решает, увидит ли их пользователь. Ваш код может пройти все проверки браузера — и всё равно не достичь цели.

Понимание этой границы — то, что отличает рабочее демо от демо, которое «работает на моей машине».

Приложение (полные исходные коды данной главы)

**sw.js**:

```
// Service Worker — один на origin.
// Обслуживает:
//   • offline-first (глава 20): cache-first / network-first.
//   • клик по SW-уведомлению (глава 33): notificationclick.
//   • закрытие SW-уведомления (глава 33): notificationclose.
//
// Один scope `/` — второй SW в том же scope невозможен.

const CACHE_APP_SHELL = 'app-shell-v1';
const CACHE_RUNTIME = 'runtime-v1';

const ASSETS = [
    '/',
    '/index.html',
    '/pkg/app.js',
    '/pkg/app_bg.wasm',
];

// ============================================================
//                    INSTALL / ACTIVATE
// ============================================================

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

// ============================================================
//                    FETCH (offline-first)
// ============================================================

self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url);

    if (url.pathname.startsWith('/api/')) {
        event.respondWith(networkFirst(event.request));
        return;
    }

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

// ============================================================
//                    NOTIFICATIONCLICK (глава 33)
// ============================================================
//
// Срабатывает, когда пользователь кликает по SW-уведомлению.
// Контекст: SW, не страница. Это ключевое отличие от page-уведомлений —
// там клик приходит в `notification.onclick` страницы.
//
// Задача: закрыть попап, отправить сообщение всем открытым вкладкам,
// сфокусировать существующую или открыть новую.

self.addEventListener('notificationclick', (event) => {
    event.notification.close();

    const tag = event.notification.tag || '';

    event.waitUntil((async () => {
        const clients = await self.clients.matchAll({
            type: 'window',
            includeUncontrolled: true,
        });

        // Оповещаем страницы о клике. Это то, чего нет у page-уведомлений:
        // событие приходит в SW и только потом — во вкладки.
        for (const client of clients) {
            client.postMessage({ type: 'notification-click', tag });
        }

        // Фокусируем первую вкладку или открываем новую.
        for (const client of clients) {
            if ('focus' in client) {
                return client.focus();
            }
        }
        if (self.clients.openWindow) {
            return self.clients.openWindow('/');
        }
    })());
});

// ============================================================
//                    NOTIFICATIONCLOSE (глава 33)
// ============================================================
//
// Пользователь закрыл попап (крестик). Тоже событие SW, не страницы.

self.addEventListener('notificationclose', (event) => {
    const tag = event.notification.tag || '';
    event.waitUntil((async () => {
        const clients = await self.clients.matchAll({
            type: 'window',
            includeUncontrolled: true,
        });
        for (const client of clients) {
            client.postMessage({ type: 'notification-close', tag });
        }
    })());
});
```

**notif_core.js**:
```
//! Page Notifications: permission, `Notification`, manager.
//!
//! `Notification` — попап поверх окна браузера. Три уровня:
//!
//!   1. Permission — `default` / `granted` / `denied`. Sticky per origin.
//!      Один раз `denied` — JS не может вызвать диалог снова.
//!
//!   2. `new Notification(title, options)` — попап, привязанный к
//!      вкладке. Умирает вместе с вкладкой. Это страничный уровень.
//!
//!   3. Service Worker `registration.showNotification()` — попап,
//!      который живёт вне вкладки. Разбирается в `notif_sw.rs`.
//!
//! ПРАВИЛО: работаем через `Reflect::get` — `web-sys::Notification`
//! требует фичи `Notification`, которой нет в Cargo.toml. Reflect
//! работает с любой версией.
//!
//! ПРАВИЛО: `Notification.permission` — статическое свойство
//! конструктора, не экземпляра. Читаем через Reflect от ctor.
//!
//! ПРАВИЛО: `requestPermission()` требует user gesture в Firefox
//! (SecurityError) и молча игнорируется в Chrome без жеста.
//! Демо вызывает только из обработчика клика.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use crate::callbacks::EventClosures;

// ============================================================
//                    PERMISSION
// ============================================================

/// Состояние разрешения на показ уведомлений.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Permission {
    /// Пользователь ещё не отвечал на запрос.
    Default,
    /// Разрешено.
    Granted,
    /// Запрещено. Sticky per origin — JS не может вызвать диалог снова.
    Denied,
    /// Значение не распознано или API недоступен.
    Unknown,
}

impl Permission {
    pub fn from_str(s: &str) -> Self {
        match s {
            "default" => Permission::Default,
            "granted" => Permission::Granted,
            "denied" => Permission::Denied,
            _ => Permission::Unknown,
        }
    }
    pub fn as_str(&self) -> &'static str {
        match self {
            Permission::Default => "default",
            Permission::Granted => "granted",
            Permission::Denied => "denied",
            Permission::Unknown => "unknown",
        }
    }
    pub fn short(&self) -> &'static str {
        match self {
            Permission::Default => "не запрошено",
            Permission::Granted => "разрешено",
            Permission::Denied => "запрещено",
            Permission::Unknown => "неизвестно",
        }
    }
    /// Цветовая категория — для UI.
    pub fn tone(&self) -> &'static str {
        match self {
            Permission::Granted => "ok",
            Permission::Denied => "err",
            Permission::Default => "warn",
            Permission::Unknown => "muted",
        }
    }
}

/// Возвращает конструктор `Notification` — или ошибку, если API
/// недоступен (secure context отсутствует, браузер не поддерживает).
fn notification_ctor() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("Notification"))
        .map_err(|e| JsValue::from_str(&format!("Reflect Notification: {e:?}")))?;
    if ctor.is_undefined() || ctor.is_null() {
        return Err(JsValue::from_str(
            "Notification API недоступен (нужен secure context и поддержка браузера)",
        ));
    }
    Ok(ctor)
}

/// true, если `Notification` доступен.
pub fn is_supported() -> bool {
    return notification_ctor().is_ok();
}

/// Читает `Notification.permission` — статическое свойство конструктора.
pub fn permission() -> Permission {
    let ctor = match notification_ctor() {
        Ok(c) => c,
        Err(_) => return Permission::Unknown,
    };
    let p = match js_sys::Reflect::get(&ctor, &JsValue::from_str("permission")) {
        Ok(v) => v,
        Err(_) => return Permission::Unknown,
    };
    let s = p.as_string().unwrap_or_else(|| "unknown".to_string());
    return Permission::from_str(&s);
}

/// Запрашивает разрешение. Возвращает новое состояние.
///
/// **Требует user gesture.** В Firefox без жеста бросает `SecurityError`.
/// В Chrome без жеста промис не резолвится (молчание).
///
/// **Если permission === "denied"** — возвращает `Denied` без запроса.
/// JS не может повторно вызвать диалог. Пользователь должен зайти
/// в настройки сайта вручную.
pub async fn request_permission() -> Result<Permission, JsValue> {
    let ctor = notification_ctor()?;
    let f = js_sys::Reflect::get(&ctor, &JsValue::from_str("requestPermission"))
        .map_err(|e| JsValue::from_str(&format!("Reflect requestPermission: {e:?}")))?;
    let f: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("requestPermission не функция"))?;
    let result = f
        .call0(&ctor)
        .map_err(|e| JsValue::from_str(&format!("requestPermission: {e:?}")))?;
    if let Some(p) = result.dyn_ref::<js_sys::Promise>() {
        let awaited = JsFuture::from(p.clone()).await?;
        let s = awaited.as_string().unwrap_or_else(|| "unknown".to_string());
        return Ok(Permission::from_str(&s));
    }
    if let Some(s) = result.as_string() {
        return Ok(Permission::from_str(&s));
    }
    Ok(Permission::Unknown)
}

// ============================================================
//                    NOTIFICATION HANDLE
// ============================================================

/// Дескриптор активного уведомления.
pub struct NotificationHandle {
    raw: JsValue,
    _closures: EventClosures,
}

impl NotificationHandle {
    pub fn raw(&self) -> &JsValue {
        return &self.raw;
    }

    /// Закрывает уведомление. Идемпотентно.
    pub fn close(&self) {
        if let Ok(f) = js_sys::Reflect::get(&self.raw, &JsValue::from_str("close"))
            .and_then(|v| v.dyn_into::<js_sys::Function>())
        {
            let _ = f.call0(&self.raw);
        }
    }
}

// ============================================================
//                    NOTIFY OPTIONS
// ============================================================

#[derive(Clone, Debug, Default)]
pub struct NotifyOptions {
    pub title: String,
    pub body: String,
    /// Идентификатор для группировки. Уведомление с тем же `tag`
    /// заменяет предыдущее, а не добавляется в очередь.
    pub tag: String,
    /// URL иконки. Может быть проигнорирован платформой.
    pub icon: Option<String>,
    /// Не закрывать автоматически — ждать действия пользователя.
    /// Поддерживается только на десктопе.
    pub require_interaction: bool,
}

// ============================================================
//                    NOTIFICATION MANAGER
// ============================================================

/// Менеджер активных page-уведомлений.
#[derive(Clone)]
pub struct NotificationManager {
    active: Rc<RefCell<Vec<NotificationHandle>>>,
}

impl NotificationManager {
    pub fn new() -> Self {
        return Self {
            active: Rc::new(RefCell::new(Vec::new())),
        };
    }

    pub fn count(&self) -> usize {
        return self.active.borrow().len();
    }

    pub fn close_all(&self) {
        let mut list = self.active.borrow_mut();
        for handle in list.iter() {
            handle.close();
        }
        list.clear();
    }

    /// Показывает уведомление. `on_event` вызывается на `"show"`,
    /// `"click"`, `"close"`, `"error"`.
    ///
    /// **Важно:** успешный возврат `Ok(())` означает только то, что
    /// конструктор не бросил исключение. Реальный показ ОС
    /// подтверждается событием `"show"` в `on_event`. Диагностируйте
    /// отсутствие события через таймер (см. `app.rs`).
    pub fn show<F>(
        &self,
        opts: NotifyOptions,
        on_event: F,
    ) -> Result<(), JsValue>
    where
        F: FnMut(&str) + 'static,
    {
        let ctor = notification_ctor()?;

        let js_opts = build_notify_options(&opts);

        let args = js_sys::Array::new();
        args.push(&JsValue::from_str(&opts.title));
        args.push(&js_opts);

        let notification = js_sys::Reflect::construct(
            &ctor.dyn_into::<js_sys::Function>().map_err(|_| {
                JsValue::from_str("Notification не конструктор")
            })?,
            &args,
        )
        .map_err(|e| JsValue::from_str(&format!("new Notification: {e:?}")))?;

        let mut closures = EventClosures::new();
        let on_event = Rc::new(RefCell::new(Box::new(on_event)
            as Box<dyn FnMut(&str)>));

        for (event_name, label) in [
            ("show", "show"),
            ("click", "click"),
            ("close", "close"),
            ("error", "error"),
        ] {
            let cb = on_event.clone();
            let label_s = label.to_string();
            let closure = Closure::<dyn FnMut(web_sys::Event)>::new(
                move |_e: web_sys::Event| {
                    let mut f = cb.borrow_mut();
                    f(&label_s);
                },
            );
            let _ = js_sys::Reflect::set(
                &notification,
                &JsValue::from_str(&format!("on{event_name}")),
                closure.as_ref().unchecked_ref(),
            );
            closures.push(closure);
        }

        self.active.borrow_mut().push(NotificationHandle {
            raw: notification,
            _closures: closures,
        });
        Ok(())
    }
}

impl Default for NotificationManager {
    fn default() -> Self {
        return Self::new();
    }
}

// ============================================================
//                    HELPERS
// ============================================================

fn build_notify_options(opts: &NotifyOptions) -> JsValue {
    let obj = js_sys::Object::new();
    let _ = js_sys::Reflect::set(
        &obj,
        &JsValue::from_str("body"),
        &JsValue::from_str(&opts.body),
    );
    if !opts.tag.is_empty() {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("tag"),
            &JsValue::from_str(&opts.tag),
        );
    }
    if let Some(icon) = &opts.icon {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("icon"),
            &JsValue::from_str(icon),
        );
    }
    if opts.require_interaction {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("requireInteraction"),
            &JsValue::TRUE,
        );
    }
    return obj.into();
}

// ============================================================
//                    ДИАГНОСТИКА
// ============================================================
//
// Зачем: `new Notification()` не бросает исключение, даже если попап
// фактически не отображается (DND, настройки ОС/браузера, headless,
// отсутствие доступа к каналу уведомлений). Этот блок собирает
// снимок окружения, по которому можно понять, где именно теряется
// уведомление.

fn js_get(obj: &JsValue, key: &str) -> Option<JsValue> {
    js_sys::Reflect::get(obj, &JsValue::from_str(key)).ok()
}

fn js_get_str(obj: &JsValue, key: &str) -> Option<String> {
    js_get(obj, key).and_then(|v| v.as_string())
}

fn js_get_bool(obj: &JsValue, key: &str) -> Option<bool> {
    js_get(obj, key).and_then(|v| v.as_bool())
}

fn yes_no(b: bool) -> String {
    if b { "yes".into() } else { "no".into() }
}

/// `navigator.permissions.query({name: 'notifications'})` → `.state`.
///
/// Отличие от `Notification.permission`: Permissions API умеет
/// возвращать не только `granted`/`denied`, но и `prompt`, и умеет
/// подписываться на изменения через `.onchange`. Если API не
/// поддерживается — вернёт `None`.
pub async fn permission_via_permissions_api() -> Option<String> {
    let global = js_sys::global();
    let nav = js_get(&global, "navigator")?;
    let permissions = js_get(&nav, "permissions")?;
    if permissions.is_undefined() || permissions.is_null() {
        return None;
    }
    let query = js_get(&permissions, "query")?;
    let query: js_sys::Function = query.dyn_into().ok()?;
    let desc = js_sys::Object::new();
    js_sys::Reflect::set(
        &desc,
        &JsValue::from_str("name"),
        &JsValue::from_str("notifications"),
    )
    .ok()?;
    let promise = query.call1(&permissions, &desc).ok()?;
    let promise: js_sys::Promise = promise.dyn_into().ok()?;
    let result = JsFuture::from(promise).await.ok()?;
    js_get_str(&result, "state")
}

/// Собирает снимок окружения. Возвращает плоский список (ключ, значение)
/// — удобно для рендера таблицей.
pub async fn collect_diagnostics() -> Vec<(String, String)> {
    let mut out: Vec<(String, String)> = Vec::new();
    let global = js_sys::global();

    // --- secure context ---
    out.push((
        "isSecureContext".into(),
        yes_no(js_get_bool(&global, "isSecureContext").unwrap_or(false)),
    ));

    // --- Notification API ---
    out.push(("Notification API".into(), yes_no(is_supported())));
    out.push((
        "Notification.permission (legacy)".into(),
        permission().as_str().into(),
    ));

    // --- Permissions API (более надёжный источник) ---
    let perm_api = permission_via_permissions_api().await;
    out.push((
        "Permissions API (notifications)".into(),
        perm_api.clone().unwrap_or_else(|| "n/a".into()),
    ));

    // --- maxActions: если undefined — вероятно, очень старый движок ---
    let ma = notification_ctor()
        .ok()
        .and_then(|c| js_get(&c, "maxActions").and_then(|v| v.as_f64()));
    out.push((
        "Notification.maxActions".into(),
        ma.map(|n| (n as i64).to_string())
            .unwrap_or_else(|| "n/a".into()),
    ));

    // --- navigator ---
    if let Some(nav) = js_get(&global, "navigator") {
        let ua = js_get_str(&nav, "userAgent").unwrap_or_default();
        out.push(("navigator.userAgent".into(), ua.clone()));
        out.push((
            "navigator.platform".into(),
            js_get_str(&nav, "platform").unwrap_or_default(),
        ));
        out.push((
            "navigator.language".into(),
            js_get_str(&nav, "language").unwrap_or_default(),
        ));
        out.push((
            "navigator.webdriver".into(),
            yes_no(js_get_bool(&nav, "webdriver").unwrap_or(false)),
        ));
        let ua_lower = ua.to_lowercase();
        out.push((
            "UA содержит 'headless'".into(),
            yes_no(ua_lower.contains("headless")),
        ));
        out.push((
            "UA содержит 'mobile' или 'android'".into(),
            yes_no(ua_lower.contains("mobile") || ua_lower.contains("android")),
        ));
    }

    // --- document ---
    if let Some(doc) = js_get(&global, "document") {
        out.push((
            "document.visibilityState".into(),
            js_get_str(&doc, "visibilityState").unwrap_or_default(),
        ));
        let has_focus = js_get(&doc, "hasFocus")
            .and_then(|f| f.dyn_into::<js_sys::Function>().ok())
            .and_then(|f| f.call0(&doc).ok())
            .and_then(|v| v.as_bool())
            .unwrap_or(false);
        out.push(("document.hasFocus()".into(), yes_no(has_focus)));
    }

    // --- Service Worker ---
    out.push((
        "Service Worker API".into(),
        yes_no(crate::notif_sw::is_supported()),
    ));

    match crate::notif_sw::get_registration().await {
        Ok(Some(reg)) => {
            out.push(("SW registration".into(), "present".into()));
            for key in ["active", "installing", "waiting"] {
                if let Some(v) = js_get(&reg, key) {
                    if !v.is_null() && !v.is_undefined() {
                        let state = js_get_str(&v, "state").unwrap_or_default();
                        out.push((format!("SW.{key}.state"), state));
                    }
                }
            }
            match crate::notif_sw::count_via_registration(&reg).await {
                Ok(n) => out.push((
                    "SW getNotifications().length".into(),
                    n.to_string(),
                )),
                Err(e) => out.push((
                    "SW getNotifications()".into(),
                    format!("error: {}", e.as_string().unwrap_or_default()),
                )),
            }
        }
        Ok(None) => out.push((
            "SW registration".into(),
            "none — нажмите «Регистрировать SW»".into(),
        )),
        Err(e) => out.push((
            "SW registration".into(),
            format!("error: {}", e.as_string().unwrap_or_default()),
        )),
    }

    // --- Итоговая подсказка ---
    let mut hints: Vec<&'static str> = Vec::new();
    if !js_get_bool(&global, "isSecureContext").unwrap_or(false) {
        hints.push("Нет secure context — Notification API не работает.");
    }
    if !is_supported() {
        hints.push("Notification API недоступен в этом браузере/режиме.");
    }
    if permission() != Permission::Granted {
        hints.push("Разрешение не granted — попап не покажется.");
    }
    if js_get_bool(&global, "isSecureContext").is_none() {
        hints.push("Нет isSecureContext — вероятно, очень старый движок.");
    }
    if let Some(nav) = js_get(&global, "navigator") {
        if js_get_bool(&nav, "webdriver").unwrap_or(false) {
            hints.push(
                "navigator.webdriver = true (автотест/Selenium). \
                 Некоторые ОС глушат уведомления в этом режиме.",
            );
        }
        let ua = js_get_str(&nav, "userAgent").unwrap_or_default().to_lowercase();
        if ua.contains("headless") {
            hints.push(
                "Headless-браузер — попап может не отображаться ОС.",
            );
        }
    }
    hints.push(
        "Проверьте ОС: режим «Не беспокоить» / Focus / уведомления \
         для браузера / канал звука.",
    );

    out.push(("— подсказки —".into(), String::new()));
    for h in hints {
        out.push(("•".into(), h.to_string()));
    }

    out
}
```


**notif_sw.js**:
```
//! Service Worker notifications: `registration.showNotification()`.
//!
//! Ключевое отличие от page-notification: попап, показанный из SW,
//! **живёт вне вкладки**. Закрыл вкладку — попап остался. Клик по нему
//! приходит в SW событие `notificationclick`, а не в страницу.
//!
//! Это основа push-уведомлений: сервер шлёт push на SW, SW показывает
//! уведомление, пользователь кликает, SW получает управление.
//!
//! ПРАВИЛО: регистрация SW требует отдельный JS-файл (`web/sw.js`).
//! Путь относительный, scope ограничен директорией файла.
//!
//! ПРАВИЛО: `showNotification()` возвращает `Promise<void>`.
//!
//! ПРАВИЛО: `JsFuture::from(promise).await?` — результат всегда
//! `JsValue`. Явная аннотация `let result: JsValue` помогает
//! rust-analyzer вывести тип на host-таргете.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use crate::notif_core::NotifyOptions;

// ============================================================
//                    ДОСТУП
// ============================================================

/// Возвращает `navigator.serviceWorker` — или ошибку.
fn service_worker() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("Reflect navigator: {e:?}")))?;
    let sw = js_sys::Reflect::get(&nav, &JsValue::from_str("serviceWorker"))
        .map_err(|e| JsValue::from_str(&format!("Reflect serviceWorker: {e:?}")))?;
    if sw.is_undefined() || sw.is_null() {
        return Err(JsValue::from_str(
            "Service Worker API недоступен (нужен secure context)",
        ));
    }
    return Ok(sw);
}

/// true, если Service Worker API доступен.
pub fn is_supported() -> bool {
    return service_worker().is_ok();
}

// ============================================================
//                    РЕГИСТРАЦИЯ
// ============================================================

/// Регистрирует Service Worker по указанному URL.
///
/// Возвращает `ServiceWorkerRegistration` (как `JsValue`). Идемпотентно:
/// повторный вызов с тем же URL возвращает ту же регистрацию.
pub async fn register(url: &str) -> Result<JsValue, JsValue> {
    let sw = service_worker()?;
    let register_fn = js_sys::Reflect::get(&sw, &JsValue::from_str("register"))
        .map_err(|e| JsValue::from_str(&format!("Reflect register: {e:?}")))?;
    let register_fn: js_sys::Function = register_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("register не функция"))?;
    let promise = register_fn
        .call1(&sw, &JsValue::from_str(url))
        .map_err(|e| JsValue::from_str(&format!("register: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("register не вернул Promise"))?;
    JsFuture::from(promise).await
}

/// Возвращает существующую регистрацию, если есть.
///
/// `getRegistration()` резолвится с `ServiceWorkerRegistration | undefined`.
/// `undefined` означает, что SW для этого scope не зарегистрирован.
pub async fn get_registration() -> Result<Option<JsValue>, JsValue> {
    let sw = service_worker()?;
    let get_fn = js_sys::Reflect::get(&sw, &JsValue::from_str("getRegistration"))
        .map_err(|e| JsValue::from_str(&format!("Reflect getRegistration: {e:?}")))?;
    let get_fn: js_sys::Function = get_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("getRegistration не функция"))?;
    let promise = get_fn
        .call0(&sw)
        .map_err(|e| JsValue::from_str(&format!("getRegistration: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("getRegistration не вернул Promise"))?;
    let result: JsValue = JsFuture::from(promise).await?;
    if result.is_undefined() || result.is_null() {
        return Ok(None);
    }
    return Ok(Some(result));
}

// ============================================================
//                    SHOW NOTIFICATION
// ============================================================

/// Показывает уведомление через Service Worker.
///
/// `registration.showNotification(title, options)` — попап живёт вне
/// вкладки. Даже если закрыть страницу — попап останется.
pub async fn show_via_registration(
    reg: &JsValue,
    opts: &NotifyOptions,
) -> Result<(), JsValue> {
    let show_fn = js_sys::Reflect::get(reg, &JsValue::from_str("showNotification"))
        .map_err(|e| JsValue::from_str(&format!("Reflect showNotification: {e:?}")))?;
    let show_fn: js_sys::Function = show_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("showNotification не функция"))?;

    let title_arg = JsValue::from_str(&opts.title);
    let options_arg = build_options_js(opts);
    let promise = show_fn
        .call2(reg, &title_arg, &options_arg)
        .map_err(|e| JsValue::from_str(&format!("showNotification: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("showNotification не вернул Promise"))?;
    let _result: JsValue = JsFuture::from(promise).await?;
    Ok(())
}

/// Возвращает количество активных SW-уведомлений.
///
/// `getNotifications()` возвращает все уведомления, показанные через
/// этот SW.
pub async fn count_via_registration(reg: &JsValue) -> Result<u32, JsValue> {
    let f = js_sys::Reflect::get(reg, &JsValue::from_str("getNotifications"))
        .map_err(|e| JsValue::from_str(&format!("Reflect getNotifications: {e:?}")))?;
    let f: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications не функция"))?;
    let promise = f
        .call0(reg)
        .map_err(|e| JsValue::from_str(&format!("getNotifications: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications не вернул Promise"))?;
    let result: JsValue = JsFuture::from(promise).await?;
    let arr: js_sys::Array = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications вернул не Array"))?;
    return Ok(arr.length());
}

/// Закрывает все активные SW-уведомления.
///
/// Вызывает `.close()` на каждом объекте из `getNotifications()`.
/// Возвращает количество закрытых.
pub async fn close_all_via_registration(reg: &JsValue) -> Result<u32, JsValue> {
    let f = js_sys::Reflect::get(reg, &JsValue::from_str("getNotifications"))
        .map_err(|e| JsValue::from_str(&format!("Reflect getNotifications: {e:?}")))?;
    let f: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications не функция"))?;
    let promise = f
        .call0(reg)
        .map_err(|e| JsValue::from_str(&format!("getNotifications: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications не вернул Promise"))?;
    let result: JsValue = JsFuture::from(promise).await?;
    let arr: js_sys::Array = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("getNotifications вернул не Array"))?;

    let mut closed = 0u32;
    for i in 0..arr.length() {
        let n = arr.get(i);
        if let Ok(close_fn) = js_sys::Reflect::get(&n, &JsValue::from_str("close"))
            .and_then(|v| v.dyn_into::<js_sys::Function>())
        {
            let _ = close_fn.call0(&n);
            closed += 1;
        }
    }
    return Ok(closed);
}

// ============================================================
//                    HELPERS
// ============================================================

fn build_options_js(opts: &NotifyOptions) -> JsValue {
    let obj = js_sys::Object::new();
    let _ = js_sys::Reflect::set(
        &obj,
        &JsValue::from_str("body"),
        &JsValue::from_str(&opts.body),
    );
    if !opts.tag.is_empty() {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("tag"),
            &JsValue::from_str(&opts.tag),
        );
    }
    if let Some(icon) = &opts.icon {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("icon"),
            &JsValue::from_str(icon),
        );
    }
    if opts.require_interaction {
        let _ = js_sys::Reflect::set(
            &obj,
            &JsValue::from_str("requireInteraction"),
            &JsValue::TRUE,
        );
    }
    return obj.into();
}
```


**lib.rs**:
```
pub mod notif_core;
pub mod notif_sw;
```

**wasm_notif_core.rs**:
```
//! Wasm-тесты `notif_core`: permission-парсинг, менеджер, состояние.
//!
//! ПРАВИЛО: тесты, требующие user gesture или granted permission,
//! помечены `#[ignore]` — в headless они не работают.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::notif_core::{self, NotifyOptions, NotificationManager, Permission};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- Permission: парсинг ----------------

#[wasm_bindgen_test]
fn permission_from_str_known_values() {
    assert_eq!(Permission::from_str("default"), Permission::Default);
    assert_eq!(Permission::from_str("granted"), Permission::Granted);
    assert_eq!(Permission::from_str("denied"), Permission::Denied);
}

#[wasm_bindgen_test]
fn permission_from_str_unknown() {
    assert_eq!(Permission::from_str("garbage"), Permission::Unknown);
    assert_eq!(Permission::from_str(""), Permission::Unknown);
}

#[wasm_bindgen_test]
fn permission_as_str_roundtrip() {
    for p in [
        Permission::Default,
        Permission::Granted,
        Permission::Denied,
        Permission::Unknown,
    ] {
        assert_eq!(Permission::from_str(p.as_str()), p);
    }
}

#[wasm_bindgen_test]
fn permission_shorts() {
    assert_eq!(Permission::Default.short(), "не запрошено");
    assert_eq!(Permission::Granted.short(), "разрешено");
    assert_eq!(Permission::Denied.short(), "запрещено");
}

#[wasm_bindgen_test]
fn permission_tones() {
    assert_eq!(Permission::Granted.tone(), "ok");
    assert_eq!(Permission::Denied.tone(), "err");
    assert_eq!(Permission::Default.tone(), "warn");
    assert_eq!(Permission::Unknown.tone(), "muted");
}

// ---------------- API availability ----------------

#[wasm_bindgen_test]
fn api_supported_in_headless_chrome() {
    // В headless Chrome на localhost Notification API есть.
    assert!(notif_core::is_supported());
}

#[wasm_bindgen_test]
fn permission_returns_known_state() {
    // Не предполагаем конкретное значение — только что функция
    // не паникует и возвращает валидный вариант.
    let p = notif_core::permission();
    assert!(
        matches!(
            p,
            Permission::Default | Permission::Granted | Permission::Denied | Permission::Unknown
        ),
        "permission вернул невалидное состояние: {:?}", p,
    );
}

// ---------------- NotificationManager: локальное состояние ----------------

#[wasm_bindgen_test]
fn manager_starts_empty() {
    let m = NotificationManager::new();
    assert_eq!(m.count(), 0);
}

#[wasm_bindgen_test]
fn close_all_on_empty_is_noop() {
    let m = NotificationManager::new();
    m.close_all();
    assert_eq!(m.count(), 0);
    // Идемпотентность.
    m.close_all();
    assert_eq!(m.count(), 0);
}

#[wasm_bindgen_test]
fn manager_clone_shares_state() {
    let m1 = NotificationManager::new();
    let m2 = m1.clone();
    // Оба клона смотрят на один Vec.
    assert_eq!(m1.count(), 0);
    assert_eq!(m2.count(), 0);
}

// ---------------- NotifyOptions: значения по умолчанию ----------------

#[wasm_bindgen_test]
fn notify_options_default() {
    let opts = NotifyOptions::default();
    assert!(opts.title.is_empty());
    assert!(opts.body.is_empty());
    assert!(opts.tag.is_empty());
    assert!(opts.icon.is_none());
    assert!(!opts.require_interaction);
}

// ---------------- Show — требует разрешения ----------------

#[wasm_bindgen_test]
#[ignore = "требует granted permission — в headless недоступно"]
fn show_creates_notification() {
    let m = NotificationManager::new();
    let opts = NotifyOptions {
        title: "Test".to_string(),
        body: "Body".to_string(),
        ..Default::default()
    };
    m.show(opts, |_event| {}).expect("show");
    assert_eq!(m.count(), 1);
}
```

**wasm_notif_sw.rs**:
```
//! Wasm-тесты `notif_sw`: доступность, регистрация, showNotification.
//!
//! Регистрация Service Worker в headless-тесте требует, чтобы `sw.js`
//! был доступен по URL. Тесты, требующие реальной регистрации,
//! помечены `#[ignore]` — их надо запускать вручную после `npm start`.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::notif_sw;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn service_worker_api_supported() {
    // В headless Chrome на localhost Service Worker API есть.
    assert!(notif_sw::is_supported());
}

#[wasm_bindgen_test]
#[ignore = "требует загруженный sw.js — тест запускается после npm start"]
async fn get_registration_initially_none() {
    let reg = notif_sw::get_registration().await.expect("get_registration");
    assert!(reg.is_none(), "регистрация не должна существовать до register");
}

#[wasm_bindgen_test]
#[ignore = "требует загруженный sw.js — тест запускается после npm start"]
async fn register_and_get() {
    let reg = notif_sw::register("/sw.js").await.expect("register");
    assert!(reg.is_object());
    let reg2 = notif_sw::get_registration().await.expect("get_registration");
    assert!(reg2.is_some());
}
```

**app.rs**:
```
//! Демо Главы 33: Notifications API.
//!
//! Три блока:
//!   1. Permission — badge, кнопка запроса. Auto-refresh при focus.
//!   2. Page notification — new Notification(), события в логе.
//!   3. SW notification — registration.showNotification(), живёт вне
//!      вкладки. Клик по SW-уведомлению приходит в SW, который
//!      отправляет сообщение обратно в страницу.
//!
//! ПЛЮС: диагностическая панель — собирает снимок окружения и
//! предупреждает, если попап не показан (нет события `show`).
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`.
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: permission-запрос только из user gesture (клик).
//! ПРАВИЛО: перед show проверяем permission === granted.
//! ПРАВИЛО: успех конструктора ≠ успех показа. Ждём событие `show`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::callback::Timeout;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::format_str::format_str;
use crate::notif_core::{self, NotificationManager, NotifyOptions, Permission};
use crate::notif_sw;

const LOG_LIMIT: usize = 15;
const SW_URL: &str = "/sw.js";
/// Сколько ждать событие `show` прежде чем предупредить о проблеме.
const SHOW_TIMEOUT_MS: u32 = 2000;

const HTML: &str = r#"
<style>
  :root {
    --nf-info: #3498db; --nf-ok: #27ae60; --nf-err: #c0392b; --nf-warn: #e67e22;
    --nf-text: #222; --nf-muted: #666;
    --nf-bg: #fff; --nf-soft: #fafafa; --nf-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --nf-text: #e6e6e6; --nf-muted: #a0a0a0;
      --nf-bg: #1e1e1e; --nf-soft: #262626; --nf-border: #3a3a3a;
    }
    body { background: #121212; }
    .nf-anno { background: #1a2233; color: #9fc4ff; }
    .nf-anno code { background: rgba(52,152,219,0.2); }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .nf-log-entry { background: #1e1e1e; }
    .nf-perm.granted { background: rgba(39,174,96,0.18); color: #b8e0b8; }
    .nf-perm.denied  { background: rgba(192,57,43,0.18); color: #e0b8b8; }
    .nf-perm.warn    { background: rgba(230,126,34,0.18); color: #f0d8a0; }
    .nf-perm.muted   { background: rgba(128,128,128,0.2); color: #aaa; }
    .nf-hint { background: #232014; color: #e6c98c; border-left-color: #5a4400; }
    .nf-diag-out { background: #1a1a1a; }
    .nf-diag-row { border-bottom-color: rgba(255,255,255,0.08); }
  }
  html, body { width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--nf-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 24px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .nf-subtitle { color: var(--nf-muted); font-size: 12px; margin: 0 0 12px; }
  .nf-subtitle code { background: rgba(128,128,128,0.15);
    padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .nf-subtitle a { color: var(--nf-info); }

  .nf-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1fr) minmax(0, 1.05fr) minmax(0, 0.9fr);
  }
  @media (max-width: 1280px) { .nf-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px) { .nf-grid { grid-template-columns: 1fr; } }
  .nf-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .nf-panel {
    border: 1px solid var(--nf-border); border-radius: 8px;
    padding: 10px 12px; background: var(--nf-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0;
  }
  .nf-panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .nf-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--nf-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .nf-anno { background: #eef4ff; border-left: 3px solid var(--nf-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .nf-anno code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px; }
  .nf-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .nf-anno b { color: #c0d8ff; } }

  .nf-hint {
    background: #fff9e6; border-left: 3px solid var(--nf-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00; overflow-wrap: anywhere;
  }
  .nf-hint b { color: #5a4400; }

  .nf-form-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0; }
  input[type="text"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--nf-bg); color: var(--nf-text);
    font-family: ui-monospace, monospace;
    flex: 1 1 100px; min-width: 0;
  }
  input[type="text"]:focus {
    outline: none; border-color: var(--nf-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  label.nf-check {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--nf-muted); cursor: pointer;
    user-select: none; white-space: nowrap; flex-shrink: 0;
  }
  input[type="checkbox"] { margin: 0; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--nf-soft); color: var(--nf-text);
    cursor: pointer; transition: background 0.15s;
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--nf-info); outline-offset: 2px; }
  button.nf-primary { background: #eaf4ff; border-color: var(--nf-info); color: #1f4e79; }
  button.nf-success { background: #e8f8ed; border-color: var(--nf-ok); color: #1e8449; }
  button.nf-danger  { background: #fdedec; border-color: var(--nf-err); color: #922b21; }

  .nf-perm {
    display: inline-block; padding: 3px 10px; border-radius: 10px;
    font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700;
    margin-left: 6px;
  }
  .nf-perm.granted { background: #e8f8ed; color: #1e8449; }
  .nf-perm.denied  { background: #fdedec; color: #922b21; }
  .nf-perm.warn    { background: #fff3cd; color: #7d5a00; }
  .nf-perm.muted   { background: #f0f0f0; color: #666; }

  .nf-status {
    display: flex; align-items: center; gap: 6px;
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--nf-soft); border-left: 3px solid #ccc;
    color: var(--nf-muted);
    min-height: 22px; overflow-wrap: anywhere; word-break: break-word;
  }
  .nf-status.ok   { background: #e8f8ed; border-color: var(--nf-ok); color: #1e8449; }
  .nf-status.err  { background: #fdedec; border-color: var(--nf-err); color: #922b21; }
  .nf-status.busy { background: #fff9e6; border-color: var(--nf-warn); color: #7d5a00; }

  .nf-count {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    background: #eef4ff; color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
    margin-left: auto;
  }
  @media (prefers-color-scheme: dark) {
    .nf-count { background: #2a3a4a; color: #cfe0ff; }
  }

  .nf-log {
    border: 1px solid var(--nf-border); border-radius: 5px;
    background: var(--nf-soft); max-height: 500px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .nf-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .nf-log-entry { display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent; }
  .nf-log-entry.ok    { border-left-color: var(--nf-ok); }
  .nf-log-entry.err   { border-left-color: var(--nf-err); }
  .nf-log-entry.info  { border-left-color: var(--nf-info); }
  .nf-log-entry.warn  { border-left-color: var(--nf-warn); }
  .nf-log-entry .t { color: var(--nf-muted); flex-shrink: 0; }
  .nf-log-entry .m { color: var(--nf-text); flex: 1; overflow-wrap: anywhere; }
  .nf-log-entry.err .m { color: #922b21; }
  .nf-log-entry.warn .m { color: #7d5a00; }

  /* Диагностика */
  .nf-diag-panel { margin-top: 12px; }
  .nf-diag-out {
    background: var(--nf-soft); border: 1px solid var(--nf-border);
    border-radius: 5px; padding: 8px 12px; margin-top: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    max-height: 360px; overflow-y: auto;
    color: var(--nf-text);
  }
  .nf-diag-row {
    display: grid; grid-template-columns: minmax(200px, 42%) 1fr;
    gap: 8px; padding: 2px 0;
    border-bottom: 1px dashed rgba(128,128,128,0.15);
  }
  .nf-diag-row .k { color: var(--nf-muted); overflow-wrap: anywhere; }
  .nf-diag-row .v { color: var(--nf-text); overflow-wrap: anywhere; }
</style>

<h1>Глава 33: Notifications API</h1>
<p class="nf-subtitle">
  Permission, page-уведомления, Service Worker уведомления.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="nf-grid">

  <!-- ============ Колонка 1: Permission ============ -->
  <div class="nf-col">
    <div class="nf-panel">
      <h2 class="nf-panel-title">
        <span class="nf-num">1</span>Permission
      </h2>
      <div class="nf-anno">
        Разрешение запрашивается один раз и <b>sticky per origin</b>.
        После <code>denied</code> JS не может вызвать диалог — только
        пользователь через настройки браузера.<br>
        <b>Focus-эффект:</b> если поменять статус в настройках, вернитесь
        на эту вкладку — badge обновится сам.
      </div>
      <div class="nf-form-row">
        <span>Статус:</span>
        <span id="nf-perm" class="nf-perm muted">…</span>
      </div>
      <div class="nf-form-row">
        <button id="nf-request" class="nf-primary">Запросить разрешение</button>
        <button id="nf-refresh-perm">↻ Перечитать</button>
      </div>
      <div id="nf-perm-status" class="nf-status">готово</div>
    </div>
  </div>

  <!-- ============ Колонка 2: Page notification ============ -->
  <div class="nf-col">
    <div class="nf-panel">
      <h2 class="nf-panel-title">
        <span class="nf-num">2</span>Page notification
      </h2>
      <div class="nf-anno">
        <code>new Notification(title, options)</code>.
        Живёт, пока открыта вкладка. Закроете вкладку — попап исчезнет.
        События <code>show/click/close</code> приходят в <b>эту страницу</b>.<br>
        <b>Важно:</b> статус «показано» — только после события <code>show</code>.
        Если за 2с событие не пришло — попап, вероятно, не отображён ОС.
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-page-title" value="Page notification">
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-page-body" value="Живёт вместе с вкладкой">
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-page-tag" placeholder="tag (для замены)">
        <label class="nf-check" title="Не закрывать автоматически — ждать действия пользователя">
          <input type="checkbox" id="nf-page-reqint"> requireInteraction
        </label>
      </div>
      <div class="nf-form-row">
        <button id="nf-page-show" class="nf-success">📢 Показать</button>
        <button id="nf-page-close-all" class="nf-danger">✕ Закрыть все</button>
        <span class="nf-count" id="nf-page-count" title="handles уведомлений, созданных в этой сессии">0</span>
      </div>
      <div id="nf-page-status" class="nf-status">готово</div>
    </div>
  </div>

  <!-- ============ Колонка 3: SW notification ============ -->
  <div class="nf-col">
    <div class="nf-panel">
      <h2 class="nf-panel-title">
        <span class="nf-num">3</span>Service Worker notification
      </h2>
      <div class="nf-anno">
        <code>registration.showNotification()</code>.
        Живёт <b>вне вкладки</b> — закройте страницу, попап останется.
        Клик по нему приходит в <b>SW</b> (не в страницу). SW отправляет
        сообщение обратно через <code>clients.postMessage</code> —
        увидите его в логе.
      </div>
      <div class="nf-hint">
        <b>Проверьте:</b> покажите SW-уведомление, закройте эту вкладку.
        Попап останется. Клик по нему — откроет вкладку заново, и в логе
        появится сообщение от SW.
      </div>
      <div class="nf-form-row">
        <span>SW:</span>
        <span id="nf-sw-status" class="nf-perm muted">не зарегистрирован</span>
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-sw-title" value="SW notification">
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-sw-body" value="Переживёт закрытие вкладки">
      </div>
      <div class="nf-form-row">
        <input type="text" id="nf-sw-tag" placeholder="tag (для замены)">
        <label class="nf-check" title="Не закрывать автоматически">
          <input type="checkbox" id="nf-sw-reqint"> requireInteraction
        </label>
      </div>
      <div class="nf-form-row">
        <button id="nf-sw-register" class="nf-primary">Регистрировать SW</button>
        <button id="nf-sw-show" class="nf-success" disabled>📢 Показать</button>
      </div>
      <div class="nf-form-row">
        <button id="nf-sw-list" disabled>↻ Список</button>
        <button id="nf-sw-close-all" class="nf-danger" disabled>✕ Закрыть все</button>
        <span class="nf-count" id="nf-sw-count" title="активных SW-уведомлений">—</span>
      </div>
      <div id="nf-sw-live" class="nf-status">готово</div>
    </div>
  </div>

  <!-- ============ Колонка 4: Журнал ============ -->
  <div class="nf-col">
    <div class="nf-panel">
      <h2 class="nf-panel-title">Журнал</h2>
      <div id="nf-log" class="nf-log">
        <div class="nf-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

</div>

<!-- ============ Диагностика (на всю ширину) ============ -->
<div class="nf-panel nf-diag-panel">
  <h2 class="nf-panel-title">🔍 Диагностика окружения</h2>
  <div class="nf-anno">
    Если попап «показан», но не виден — соберите диагностику. Типичные причины:
    системный режим <b>«Не беспокоить»</b> / Focus, настройки уведомлений для
    браузера (silent-канал), headless-браузер, отсутствие фокуса на ОС,
    политики корпоративного устройства. Панель также проверяет
    <code>Permissions API</code> (более надёжен, чем
    <code>Notification.permission</code>) и состояние SW.
  </div>
  <div class="nf-form-row">
    <button id="nf-diag-collect" class="nf-primary">🔍 Собрать диагностику</button>
    <button id="nf-diag-test" class="nf-success">🔔 Тестовое page-уведомление (requireInteraction, отдельный tag)</button>
  </div>
  <div id="nf-diag-out" class="nf-diag-out">
    <div class="nf-log-empty">нажмите «Собрать диагностику»</div>
  </div>
</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info, Warn }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct Ui {
    perm: web_sys::Element,
    perm_status: web_sys::Element,
    btn_request: web_sys::HtmlButtonElement,
    btn_refresh_perm: web_sys::HtmlButtonElement,

    page_title: web_sys::HtmlInputElement,
    page_body: web_sys::HtmlInputElement,
    page_tag: web_sys::HtmlInputElement,
    page_reqint: web_sys::HtmlInputElement,
    btn_page_show: web_sys::HtmlButtonElement,
    btn_page_close_all: web_sys::HtmlButtonElement,
    page_status: web_sys::Element,
    page_count: web_sys::Element,

    sw_status: web_sys::Element,
    sw_title: web_sys::HtmlInputElement,
    sw_body: web_sys::HtmlInputElement,
    sw_tag: web_sys::HtmlInputElement,
    sw_reqint: web_sys::HtmlInputElement,
    btn_sw_register: web_sys::HtmlButtonElement,
    btn_sw_show: web_sys::HtmlButtonElement,
    btn_sw_list: web_sys::HtmlButtonElement,
    btn_sw_close_all: web_sys::HtmlButtonElement,
    sw_live: web_sys::Element,
    sw_count: web_sys::Element,

    log: web_sys::Element,

    btn_diag_collect: web_sys::HtmlButtonElement,
    btn_diag_test: web_sys::HtmlButtonElement,
    diag_out: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    page_manager: NotificationManager,
    sw_registration: Rc<RefCell<Option<JsValue>>>,
    last_permission: Rc<Cell<Permission>>,
    ui: Ui,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        perm: get_el("nf-perm")?,
        perm_status: get_el("nf-perm-status")?,
        btn_request: get_button("nf-request")?,
        btn_refresh_perm: get_button("nf-refresh-perm")?,

        page_title: get_input("nf-page-title")?,
        page_body: get_input("nf-page-body")?,
        page_tag: get_input("nf-page-tag")?,
        page_reqint: get_input("nf-page-reqint")?,
        btn_page_show: get_button("nf-page-show")?,
        btn_page_close_all: get_button("nf-page-close-all")?,
        page_status: get_el("nf-page-status")?,
        page_count: get_el("nf-page-count")?,

        sw_status: get_el("nf-sw-status")?,
        sw_title: get_input("nf-sw-title")?,
        sw_body: get_input("nf-sw-body")?,
        sw_tag: get_input("nf-sw-tag")?,
        sw_reqint: get_input("nf-sw-reqint")?,
        btn_sw_register: get_button("nf-sw-register")?,
        btn_sw_show: get_button("nf-sw-show")?,
        btn_sw_list: get_button("nf-sw-list")?,
        btn_sw_close_all: get_button("nf-sw-close-all")?,
        sw_live: get_el("nf-sw-live")?,
        sw_count: get_el("nf-sw-count")?,

        log: get_el("nf-log")?,

        btn_diag_collect: get_button("nf-diag-collect")?,
        btn_diag_test: get_button("nf-diag-test")?,
        diag_out: get_el("nf-diag-out")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        page_manager: NotificationManager::new(),
        sw_registration: Rc::new(RefCell::new(None)),
        last_permission: Rc::new(Cell::new(Permission::Unknown)),
        ui,
    });

    app.render_log();

    if !notif_core::is_supported() {
        app.push_log(LogKind::Err,
            "Notification API недоступен — нужен secure context".to_string());
    } else {
        app.push_log(LogKind::Info, "Notification API доступен".to_string());
    }
    if !notif_sw::is_supported() {
        app.push_log(LogKind::Warn,
            "Service Worker API недоступен — SW-уведомления не заработают".to_string());
    } else {
        app.push_log(LogKind::Info, "Service Worker API доступен".to_string());
    }

    refresh_permission(&app, true);

    // ------------- Permission: запрос -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_request, move |_| {
            let app_in = app_c.clone();
            set_status(&app_in.ui.perm_status, "busy", "запрос разрешения…");
            spawn_local(async move {
                match notif_core::request_permission().await {
                    Ok(p) => {
                        refresh_permission(&app_in, true);
                        let msg = format_str(format_args!(
                            "requestPermission → {}", p.as_str(),
                        ));
                        app_in.push_log(LogKind::Ok, msg.clone());
                        set_status(&app_in.ui.perm_status, "ok", &msg);
                    }
                    Err(e) => {
                        let msg = format_str(format_args!(
                            "requestPermission: {}", short_error(&e),
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.perm_status, "err", &msg);
                    }
                }
            });
        });
    }

    // ------------- Permission: перечитать -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_refresh_perm, move |_| {
            refresh_permission(&app_c, false);
            set_status(&app_c.ui.perm_status, "ok", "перечитано");
        });
    }

    // ------------- window.focus → пересчёт permission -------------
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
            refresh_permission(&app_c, false);
        });
        if let Some(win) = web_sys::window() {
            win.add_event_listener_with_callback("focus", cb.as_ref().unchecked_ref())?;
        }
        cb.forget();
    }

    // ------------- SW message → лог -------------
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::MessageEvent)>::new(
            move |ev: web_sys::MessageEvent| {
                let data = ev.data();
                let ty = js_sys::Reflect::get(&data, &JsValue::from_str("type"))
                    .ok().and_then(|v| v.as_string()).unwrap_or_default();
                let tag = js_sys::Reflect::get(&data, &JsValue::from_str("tag"))
                    .ok().and_then(|v| v.as_string()).unwrap_or_default();
                let tag_label = if tag.is_empty() { "—".to_string() } else { tag };
                match ty.as_str() {
                    "notification-click" => {
                        let msg = format_str(format_args!(
                            "← SW: клик по уведомлению (tag={tag_label})",
                        ));
                        app_c.push_log(LogKind::Ok, msg.clone());
                        set_status(&app_c.ui.sw_live, "ok", &msg);
                        refresh_sw_count(app_c.clone());
                    }
                    "notification-close" => {
                        let msg = format_str(format_args!(
                            "← SW: уведомление закрыто (tag={tag_label})",
                        ));
                        app_c.push_log(LogKind::Info, msg.clone());
                        refresh_sw_count(app_c.clone());
                    }
                    _ => {}
                }
            },
        );
        if let Some(win) = web_sys::window() {
            if let Ok(nav) = win.navigator().dyn_into::<web_sys::Navigator>() {
                let sw_val = js_sys::Reflect::get(&nav, &JsValue::from_str("serviceWorker"))
                    .unwrap_or(JsValue::UNDEFINED);
                if !sw_val.is_undefined() {
                    if let Ok(target) = sw_val.dyn_into::<web_sys::EventTarget>() {
                        target.add_event_listener_with_callback(
                            "message",
                            cb.as_ref().unchecked_ref(),
                        )?;
                    }
                }
            }
        }
        cb.forget();
    }

    // ------------- Page: показать -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_page_show, move |_| {
            let app_in = app_c.clone();
            let title = app_in.ui.page_title.value();
            let body = app_in.ui.page_body.value();
            let tag = app_in.ui.page_tag.value();
            let reqint = app_in.ui.page_reqint.checked();
            show_page_notification(app_in, title, body, tag, reqint);
        });
    }

    // ------------- Page: закрыть все -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_page_close_all, move |_| {
            let app_in = app_c.clone();
            let n = app_in.page_manager.count();
            app_in.page_manager.close_all();
            update_page_count(&app_in);
            let msg = format_str(format_args!("закрыто {n} handles page-уведомлений"));
            app_in.push_log(LogKind::Info, msg.clone());
            set_status(&app_in.ui.page_status, "ok", &msg);
        });
    }

    // ------------- SW: регистрация -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_sw_register, move |_| {
            let app_in = app_c.clone();
            app_in.ui.btn_sw_register.set_disabled(true);
            set_status(&app_in.ui.sw_live, "busy", "регистрация SW…");
            app_in.push_log(LogKind::Info, "register('/sw.js')".to_string());
            spawn_local(async move {
                match notif_sw::register(SW_URL).await {
                    Ok(reg) => {
                        *app_in.sw_registration.borrow_mut() = Some(reg);
                        set_sw_status_registered(&app_in);
                        set_status(&app_in.ui.sw_live, "ok", "SW зарегистрирован");
                        app_in.push_log(LogKind::Ok, "SW зарегистрирован".to_string());
                        app_in.ui.btn_sw_show.set_disabled(false);
                        app_in.ui.btn_sw_list.set_disabled(false);
                        app_in.ui.btn_sw_close_all.set_disabled(false);
                        app_in.ui.btn_sw_register.set_disabled(false);
                    }
                    Err(e) => {
                        let msg = format_str(format_args!(
                            "register: {}", short_error(&e),
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.sw_live, "err", &msg);
                        app_in.ui.btn_sw_register.set_disabled(false);
                    }
                }
            });
        });
    }

    // ------------- SW: показать -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_sw_show, move |_| {
            let app_in = app_c.clone();
            let title = app_in.ui.sw_title.value();
            let body = app_in.ui.sw_body.value();
            let tag = app_in.ui.sw_tag.value();
            let reqint = app_in.ui.sw_reqint.checked();
            show_sw_notification(app_in, title, body, tag, reqint);
        });
    }

    // ------------- SW: список -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_sw_list, move |_| {
            refresh_sw_count(app_c.clone());
        });
    }

    // ------------- SW: закрыть все -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_sw_close_all, move |_| {
            let app_in = app_c.clone();
            let reg = app_in.sw_registration.borrow().clone();
            let Some(reg) = reg else {
                set_status(&app_in.ui.sw_live, "err", "SW не зарегистрирован");
                return;
            };
            set_status(&app_in.ui.sw_live, "busy", "закрытие…");
            spawn_local(async move {
                match notif_sw::close_all_via_registration(&reg).await {
                    Ok(n) => {
                        let msg = format_str(format_args!("закрыто {n} SW-уведомлений"));
                        app_in.push_log(LogKind::Info, msg.clone());
                        set_status(&app_in.ui.sw_live, "ok", &msg);
                        refresh_sw_count(app_in);
                    }
                    Err(e) => {
                        let msg = format_str(format_args!(
                            "close_all: {}", short_error(&e),
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.sw_live, "err", &msg);
                    }
                }
            });
        });
    }

    // ------------- Диагностика: собрать -------------
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_diag_collect, move |_| {
            collect_and_show_diagnostics(app_c.clone());
        });
    }

    // ------------- Диагностика: тестовое уведомление -------------
    //
    // Отдельный tag, requireInteraction = true. Если оно не видно —
    // проблема почти наверняка в ОС/настройках браузера, а не в JS.
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_diag_test, move |_| {
            show_page_notification(
                app_c.clone(),
                "🔔 TEST 33 — проверка".to_string(),
                "Если попап виден — уведомления работают. Если нет — соберите диагностику.".to_string(),
                "test-33-diag".to_string(),
                true,
            );
        });
    }

    // Автопоиск существующей регистрации.
    {
        let app_c = app.clone();
        spawn_local(async move {
            match notif_sw::get_registration().await {
                Ok(Some(reg)) => {
                    *app_c.sw_registration.borrow_mut() = Some(reg);
                    set_sw_status_registered(&app_c);
                    app_c.ui.btn_sw_show.set_disabled(false);
                    app_c.ui.btn_sw_list.set_disabled(false);
                    app_c.ui.btn_sw_close_all.set_disabled(false);
                    app_c.push_log(LogKind::Info,
                        "существующая регистрация SW найдена".to_string());
                    refresh_sw_count(app_c);
                }
                Ok(None) => {
                    app_c.push_log(LogKind::Info,
                        "регистрации SW нет — нажмите «Регистрировать SW»".to_string());
                }
                Err(_) => {}
            }
        });
    }

    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn show_page_notification(
    app: Rc<App>,
    title: String,
    body: String,
    tag: String,
    require_interaction: bool,
) {
    if title.is_empty() {
        set_status(&app.ui.page_status, "err", "введите title");
        return;
    }

    // Pre-check: если permission !== granted, браузер молча не покажет.
    let p = notif_core::permission();
    if p != Permission::Granted {
        let msg = format_str(format_args!(
            "сначала разрешите уведомления — permission = {}",
            p.as_str(),
        ));
        app.push_log(LogKind::Warn, msg.clone());
        set_status(&app.ui.page_status, "err", &msg);
        return;
    }

    let opts = NotifyOptions {
        title: title.clone(),
        body,
        tag,
        icon: None,
        require_interaction,
    };

    let log_cb = app.log.clone();
    let manager_cb = app.page_manager.clone();
    let count_el = app.ui.page_count.clone();
    let status_el = app.ui.page_status.clone();

    // Флаг «получили хотя бы одно событие от этого уведомления».
    let saw_event = Rc::new(Cell::new(false));
    let saw_event_cb = saw_event.clone();

    // ВАЖНО: `title` понадобится ещё в таймауте ниже, поэтому в
    // замыкание уходит отдельный клон. Иначе borrow-of-moved.
    let title_for_event = title.clone();

    let on_event = move |event_name: &str| {
        saw_event_cb.set(true);
        let msg = format_str(format_args!(
            "page · {title_for_event} · {event_name}",
        ));
        let kind = match event_name {
            "click" => LogKind::Ok,
            "close" => LogKind::Info,
            "error" => LogKind::Err,
            _ => LogKind::Info,
        };
        push_log_static(&log_cb, kind, msg.clone());

        // Ключевое: обновляем статус по факту события.
        match event_name {
            "show" => set_status(
                &status_el,
                "ok",
                "✓ событие show получено — попап передан ОС",
            ),
            "error" => set_status(
                &status_el,
                "err",
                "✗ браузер сообщил об ошибке показа уведомления",
            ),
            _ => set_status(&status_el, kind_to_status(kind), &msg),
        }
        update_count_static(&count_el, manager_cb.count() as u32);
    };

    match app.page_manager.show(opts, on_event) {
        Ok(()) => {
            update_page_count(&app);
            app.push_log(
                LogKind::Info,
                "new Notification(...) — конструктор не бросил исключение".to_string(),
            );
            set_status(
                &app.ui.page_status,
                "busy",
                "создано — ждём событие show…",
            );

            // Если за SHOW_TIMEOUT_MS событие не пришло — предупреждаем.
            let app_t = app.clone();
            let saw_event_t = saw_event.clone();
            let title_t = title.clone();
            Timeout::new(SHOW_TIMEOUT_MS, move || {
                if !saw_event_t.get() {
                    let msg = format_str(format_args!(
                        "⚠ {SHOW_TIMEOUT_MS}мс прошло, событие show для «{title_t}» не получено — попап, вероятно, не отображён ОС",
                    ));
                    app_t.push_log(LogKind::Warn, msg.clone());
                    set_status(
                        &app_t.ui.page_status,
                        "err",
                        "нет события show — проверьте: режим «Не беспокоить»/Focus, настройки уведомлений браузера, настройки сайта. См. «Диагностику» ниже.",
                    );
                }
            })
            .forget();
        }
        Err(e) => {
            let msg = format_str(format_args!("show: {}", short_error(&e)));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.page_status, "err", &msg);
        }
    }
}

fn show_sw_notification(
    app: Rc<App>,
    title: String,
    body: String,
    tag: String,
    require_interaction: bool,
) {
    if title.is_empty() {
        set_status(&app.ui.sw_live, "err", "введите title");
        return;
    }

    let p = notif_core::permission();
    if p != Permission::Granted {
        let msg = format_str(format_args!(
            "сначала разрешите уведомления — permission = {}",
            p.as_str(),
        ));
        app.push_log(LogKind::Warn, msg.clone());
        set_status(&app.ui.sw_live, "err", &msg);
        return;
    }

    let reg = app.sw_registration.borrow().clone();
    let Some(reg) = reg else {
        set_status(&app.ui.sw_live, "err", "SW не зарегистрирован");
        return;
    };

    let opts = NotifyOptions {
        title: title.clone(),
        body,
        tag,
        icon: None,
        require_interaction,
    };

    set_status(&app.ui.sw_live, "busy", "showNotification…");
    spawn_local(async move {
        match notif_sw::show_via_registration(&reg, &opts).await {
            Ok(()) => {
                let msg = format_str(format_args!("SW · {title} · показан"));
                app.push_log(LogKind::Ok, msg.clone());
                set_status(&app.ui.sw_live, "ok", &msg);
                // Верификация через getNotifications() — реально ли
                // зарегистрировано уведомление.
                verify_sw_notification(app);
            }
            Err(e) => {
                let msg = format_str(format_args!(
                    "showNotification: {}", short_error(&e),
                ));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.sw_live, "err", &msg);
            }
        }
    });
}

/// Проверяет, что SW-уведомление действительно зарегистрировано.
fn verify_sw_notification(app: Rc<App>) {
    let reg = app.sw_registration.borrow().clone();
    let Some(reg) = reg else { return; };
    let count_el = app.ui.sw_count.clone();
    let live_el = app.ui.sw_live.clone();
    let log = app.log.clone();
    spawn_local(async move {
        match notif_sw::count_via_registration(&reg).await {
            Ok(n) => {
                update_count_static(&count_el, n);
                if n == 0 {
                    push_log_static(
                        &log,
                        LogKind::Warn,
                        "SW getNotifications() = 0 — showNotification не привёл к регистрации попапа (проверьте ОС/DND)".to_string(),
                    );
                    set_status(
                        &live_el,
                        "err",
                        "SW getNotifications() = 0 — попап не зарегистрирован",
                    );
                } else {
                    push_log_static(
                        &log,
                        LogKind::Info,
                        format_str(format_args!(
                            "SW getNotifications().length = {n} — зарегистрировано",
                        )),
                    );
                    set_status(
                        &live_el,
                        "ok",
                        &format_str(format_args!(
                            "подтверждено: активных SW-уведомлений = {n}",
                        )),
                    );
                }
            }
            Err(_) => {
                // Не критично — некоторые движки не поддерживают.
            }
        }
    });
}

fn refresh_sw_count(app: Rc<App>) {
    let reg = app.sw_registration.borrow().clone();
    let Some(reg) = reg else { return; };
    let count_el = app.ui.sw_count.clone();
    let live_el = app.ui.sw_live.clone();
    spawn_local(async move {
        match notif_sw::count_via_registration(&reg).await {
            Ok(n) => {
                update_count_static(&count_el, n);
                if live_el.text_content().unwrap_or_default().is_empty() {
                    set_status(&live_el, "ok", "обновлено");
                }
            }
            Err(_) => { /* тихо игнорируем — счётчик может не поддерживаться */ }
        }
    });
}

/// Собирает и показывает диагностику окружения.
fn collect_and_show_diagnostics(app: Rc<App>) {
    app.ui.diag_out.set_inner_html(
        "<div class=\"nf-log-empty\">сбор…</div>",
    );
    app.push_log(LogKind::Info, "сбор диагностики…".to_string());
    spawn_local(async move {
        let rows = notif_core::collect_diagnostics().await;
        let mut html = String::with_capacity(1024);
        for (k, v) in rows.iter() {
            html.push_str("<div class=\"nf-diag-row\"><span class=\"k\">");
            html.push_str(&esc_html(k));
            html.push_str("</span><span class=\"v\">");
            html.push_str(&esc_html(v));
            html.push_str("</span></div>");
        }
        app.ui.diag_out.set_inner_html(&html);
        app.push_log(
            LogKind::Ok,
            format_str(format_args!("диагностика собрана: {} строк", rows.len())),
        );
    });
}

// ============================================================
//                    UI HELPERS
// ============================================================

fn refresh_permission(app: &Rc<App>, force_log: bool) {
    let p = notif_core::permission();
    let prev = app.last_permission.get();
    app.last_permission.set(p);

    let el = &app.ui.perm;
    el.set_text_content(Some(p.short()));
    let cls = format_str(format_args!("nf-perm {}", p.tone()));
    el.set_class_name(&cls);
    let _ = el.set_attribute("title", p.as_str());

    if force_log || prev != p {
        app.push_log(
            if prev != p && !force_log { LogKind::Ok } else { LogKind::Info },
            format_str(format_args!("permission = {}", p.as_str())),
        );
    }

    let disable = p == Permission::Denied || p == Permission::Unknown;
    app.ui.btn_request.set_disabled(disable);
    if p == Permission::Denied {
        set_status(
            &app.ui.perm_status,
            "err",
            "denied — диалог больше не появится. Разрешение меняется в настройках браузера.",
        );
    } else if p == Permission::Granted {
        set_status(&app.ui.perm_status, "ok", "разрешено — уведомления работают");
    } else if p == Permission::Default {
        set_status(
            &app.ui.perm_status,
            "busy",
            "не запрошено — нажмите «Запросить разрешение»",
        );
    }
}

fn set_sw_status_registered(app: &Rc<App>) {
    let el = &app.ui.sw_status;
    el.set_text_content(Some("зарегистрирован"));
    el.set_class_name("nf-perm granted");
    let _ = el.set_attribute("title", SW_URL);
}

fn update_page_count(app: &Rc<App>) {
    let n = app.page_manager.count();
    update_count_static(&app.ui.page_count, n as u32);
}

fn update_count_static(el: &web_sys::Element, n: u32) {
    let text = format_str(format_args!("{n}"));
    el.set_text_content(Some(&text));
}

fn kind_to_status(kind: LogKind) -> &'static str {
    match kind {
        LogKind::Ok => "ok",
        LogKind::Err => "err",
        LogKind::Info => "ok",
        LogKind::Warn => "busy",
    }
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "nf-status ok",
        "err" => "nf-status err",
        "busy" => "nf-status busy",
        _ => "nf-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn push_log_static(log: &Rc<RefCell<Vec<LogEntry>>>, kind: LogKind, text: String) {
    let time = fmt_now_hms();
    let mut v = log.borrow_mut();
    v.push(LogEntry { time, text, kind });
    if v.len() > LOG_LIMIT {
        let drop_n = v.len() - LOG_LIMIT;
        v.drain(0..drop_n);
    }
}

// ============================================================
//                    APP IMPL
// ============================================================

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        push_log_static(&self.log, kind, text);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"nf-log-empty\">здесь появятся события</div>");
            return;
        }
        let mut html = String::with_capacity(512);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
                LogKind::Warn => "warn",
            };
            html.push_str("<div class=\"nf-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui.log.set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn short_error(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        let msg = clamp_first_line(&dx.message());
        return format_str(format_args!("{}: {}", dx.name(), msg));
    }
    if let Some(s) = e.as_string() {
        return clamp_first_line(&s);
    }
    if e.is_object() {
        let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
            .ok().and_then(|v| v.as_string());
        let message = js_sys::Reflect::get(e, &JsValue::from_str("message"))
            .ok().and_then(|v| v.as_string());
        match (name, message) {
            (Some(n), Some(m)) => {
                return format_str(format_args!("{n}: {}", clamp_first_line(&m)));
            }
            (Some(n), None) => return n,
            _ => {}
        }
    }
    return format_str(format_args!("{e:?}"));
}

fn clamp_first_line(s: &str) -> String {
    let first = s.lines().next().unwrap_or("").trim();
    let mut out: String = first.chars().take(160).collect();
    if first.chars().count() > 160 {
        out.push('…');
    }
    return out;
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
    ));
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;").replace('<', "&lt;").replace('>', "&gt;")
        .replace('"', "&quot;").replace('\'', "&#39;");
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window().ok_or("no window")?
        .document().ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
    get_el(id)?.dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не input")))
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    get_el(id)?.dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where F: FnMut(web_sys::MouseEvent) + 'static {
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```