# MASTG-TEST-0334: Native Code Exposed Through WebViews
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0334/

## что проверяет этот тест

Тест MASTG-TEST-0334 ищет в коде приложения использование **WebView-JavaScript bridge** (addJavascriptInterface). Это механизм, который позволяет JavaScript-коду внутри WebView вызывать Java/Kotlin-методы приложения. Если в WebView загружается контент из ненадёжного источника (например, URL из внешнего интента, или страница с XSS), а через bridge открыт доступ к чувствительным методам - злоумышленник может выполнить код в контексте приложения: прочитать файлы, отправить запросы, украсть данные из БД


## какие инструменты использую

беру apk-файл приложения и декомпилирую через jadx-gui, потом анализирую код:

```bash
jadx-gui ~/Desktop/meetway.apk
```

в jadx ищу:
1. Все классы, где встречается `WebView`
2. Вызовы `addJavascriptInterface()`
3. Аннотации `@JavascriptInterface`
4. Методы WebSettings: `setJavaScriptEnabled()`

## как провожу тест

**шаг 1 -открываю meetway.apk в jadx-gui**

**шаг 2 - ищу `addJavascriptInterface` через встроенный поиск jadx:**
  - Text Search (Ctrl+Shift+F) → `addJavascriptInterface`
  - Text Search (Ctrl+Shift+F) → `@JavascriptInterface`

**шаг 3 - ищу WebView и его настройки через встроенный поиск jadx:**
  - Text Search (Ctrl+Shift+F) → `WebView`, `webView`
  - Text Search (Ctrl+Shift+F) → `setJavaScriptEnabled`

## что нашёл

**Результат поиска `addJavascriptInterface`: ❌ НЕ НАЙДЕНО**

| Что искал                    | Нашёл?                               |
| ---------------------------- | ------------------------------------ |
| `addJavascriptInterface()`   | ❌ нет                                |
| `@JavascriptInterface`       | ❌ нет                                |
| `setJavaScriptEnabled(true)` | ✅ есть (в YandexAuthWebViewActivity) |

**Единственный WebView в приложении** - `YandexAuthWebViewActivity.kt`. Он используется для Yandex OAuth: загружает страницу `https://oauth.yandex.ru/authorize` и перехватывает access_token из URL. JavaScript включён (`setJavaScriptEnabled(true)`), но bridge (`addJavascriptInterface`) **не настроен**

## вывод

**Тест пройден ✅**



Причина: WebView-Native bridge не используется. Несмотря на то что JavaScript включён в WebView (нужно для работы OAuth-формы Яндекса), никакой Java-объект не экспортируется в JS-окружение через `addJavascriptInterface`. Атаковать bridge нечего

Уровень теста: L1, L2
Профиль: MASVS-PLATFORM

---

