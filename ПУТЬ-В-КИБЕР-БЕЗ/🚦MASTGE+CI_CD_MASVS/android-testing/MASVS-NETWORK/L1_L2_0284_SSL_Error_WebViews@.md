# MASTG-TEST-0284: Incorrect SSL Error Handling in WebViews
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0284/

## что проверяет этот тест

Тест MASTG-TEST-0284 проверяет, не переопределяет ли приложение **`onReceivedSslError()`** в `WebViewClient` с вызовом `proceed()`. Если разработчик вручную обрабатывает SSL-ошибки и говорит «продолжить загрузку» – WebView будет загружать страницы даже с невалидным сертификатом: истекшим, самоподписанным, на другой домен.

## какие инструменты использую

беру meetway.apk, через jadx-gui смотрю WebView-активности:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx (Ctrl+Shift+F) ищу:
1. `onReceivedSslError` – переопределение метода
2. `SslErrorHandler.proceed()` – принятие SSL-ошибки
3. `WebViewClient` – кастомный клиент WebView

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Text Search ищу `onReceivedSslError`, `SslErrorHandler`, `WebViewClient`**

**шаг 3 – если нахожу – проверяю, вызывается ли `proceed()` и при каких условиях**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `onReceivedSslError` | ❌ нет |
| `SslErrorHandler.proceed()` | ❌ нет |
| `SslErrorHandler` | ❌ нет |
| `WebViewClient` (кастомный) | ❌ нет |

В приложении есть только один WebView – `YandexAuthWebViewActivity`, который использует **стандартный `WebViewClient`** без переопределения `onReceivedSslError`. По умолчанию WebView `cancel()` – SSL-ошибки блокируют загрузку.

## вывод

**Тест пройден ✅**

Причина: `onReceivedSslError` не переопределён – WebView использует стандартное поведение (cancel при SSL-ошибке). Принять невалидный сертификат через WebView нельзя.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
