# MASTG-TEST-0398: References to WebViewClient URL Loading Handlers
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0398/

**безопасно ли приложение обрабатывает переходы по ссылкам внутри WebView**. Он ищет в коде реализацию `WebViewClient` и проверяет, насколько хорошо приложение контролирует, какие URL-адреса можно загружать, чтобы не допустить открытия вредоносных сайтов или подстановки опасного контента


------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

чек поиска!

- `WebViewClient` - найти все классы, наследующие этот класс
    
- `setWebViewClient` - найти места, где WebView назначается кастомный клиент
    
- `shouldOverrideUrlLoading` - найти реализации этого метода
    
- `shouldInterceptRequest` - найти реализации этого метода


------

------


### что нашёл:

**1. WebViewClient**
- inline implementation, extends WebViewClient
- setWebViewClient 

**2. shouldOverrideUrlLoading**
- перехватывает только кастомную схему `yx547697fe6e9d46ea9a7e538922d1425d://...`
- для ВСЕГО остального возвращает `false` (т.е. WebView грузит URL внутри себя)

```kotlin
override fun shouldOverrideUrlLoading(view: WebView?, request: WebResourceRequest?): Boolean {
    val url = request?.url?.toString() ?: return false
    if (url.startsWith("yx547697fe6e9d46ea9a7e538922d1425d")) {
        handleTokenFromUrl(url)
        view?.stopLoading()
        return true
    }
    return false  // ← всё остальное загружается внутри WebView
}
```

**3. shouldInterceptRequest  не реализован** 

**4. onPageStarted есть**, срабатывает на каждый загруженный URL:
- перехватывает `yx547697fe6e9d46ea9a7e538922d1425d://`
- перехватывает `oauth.yandex.ru + access_token`
- всё остальное - просто логирует

**5. JavaScriptEnabled = true** - включён (для Яндекса норм)

**6. Activity: android:exported="false"** - не дёргается извне, ок

**7. Других WebView в приложении нет** 

------

##  че не так

### Проблема 1: shouldOverrideUrlLoading не фильтрует домены

Для любого URL, кроме кастомной схемы, возвращается `false` → WebView грузит внутри себя
Если на oauth.yandex.ru появится ссылка/редирект на левый сайт (например, реклама
Яндекс.Директа) - WebView откроет её без ограничений

`onPageStarted` тоже ничего не блокирует - только перехватывает редиректы для парсинга токена

### Проблема 2: Нет allowlist доверенных доменов

Нет проверки типа:
```kotlin
val allowedHosts = listOf("oauth.yandex.ru", "passport.yandex.ru", "social.yandex.ru")
if (allowedHosts.any { host.contains(it) }) false else true
```

------

## контекст (важно)

Этот WebView живёт ТОЛЬКО на время OAuth-авторизации. После получения токена -
Activity закрывается, пользователь уходит на MainActivity. Время жизни - секунды

Риск низкий:
- Activity не экспортирована (exported=false)
- Пользователь сам заходит на oauth.yandex.ru через запуск из приложения
- Яндекс --- респектабельный провайдер, не JS-инъекции на странице

формально это нарушает требование теста, потому что нет allowlist доверенных доменов

------

##  вердикт

shouldOverrideUrlLoading есть, но allowlist доверенных доменов НЕТ
Любой URL, на который редиректит Яндекс (или если злоумышленник сможет
повлиять на страницу), будет загружен внутри WebView

**по L1:** FAIL
**по L2:** FAIL

**MASVS:** нарушение MASVS-CODE (неконтролируемая загрузка URL в WebView)
**MASWE:** MASWE-0071 (URL Loading Handlers)

------

##  что можно улучшить

1. Добавить allowlist: проверять host в shouldOverrideUrlLoading и onPageStarted
2. Для не-allowlist URL - открывать в браузере (Intent.ACTION_VIEW)
3. Как вариант - забить, если это сознательное решение, но в тесте надо указать почему

------

## Результаты SAST

### Что искал:
- WebViewClient, setWebViewClient - кастомный клиент
- shouldOverrideUrlLoading - переопределение навигации
- shouldInterceptRequest - перехват ресурсов
- onPageStarted - обработка загрузки страниц
- JavaScriptEnabled - включён ли JS
- allowlist / whitelist - проверка доменов

### Где нашёл:

**Файл:** `YandexAuthWebViewActivity.kt` (единственный в приложении)

| Метод | Статус | Описание |
|-------|--------|----------|
| setWebViewClient | ✅ | inline WebViewClient назначен |
| shouldOverrideUrlLoading | ✅ | перехватывает только кастомную схему |
| shouldInterceptRequest | ❌ | не реализован |
| onPageStarted | ✅ | ловит редиректы с токеном |
| allowlist доменов | ❌ | нет проверок, onPageStarted не блокирует |
| JavaScriptEnabled | ✅ true | включён |

### Вердикт

тест не пройден

shouldOverrideUrlLoading реализован, но не ограничивает навигацию
do доверенных доменов. Формальное нарушение MASTG-TEST-0398


