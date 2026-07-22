# MASTG-TEST-0320: WebViews Not Cleaning Up Sensitive Data
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0320/




этот тест проверяет, не остаются ли в WebView следы чувствительных данных (пароли, куки, локальное хранилище) после того, как ты закрыл приложение. Если данные не удаляются, кто-то с доступом к файловой системе телефона (или через бэкап) может их вытащить.


WebView, как и браузер, может сохранять у себя:

- **Cookies** - сессионные данные, токены.
   
- **Кеш** - локальные копии страниц и ресурсов.
   
- **localStorage / Web SQL** - данные JavaScript (могут содержать PII).
   
- **AppCache** - устаревший, но всё ещё возможный способ хранения.
   
Приложение должно **очищать** эти данные, когда они больше не нужны, особенно после выхода из аккаунта или закрытия экрана с WebView. Тест проверяет, вызываются ли соответствующие методы очистки:

- `CookieManager.getInstance().removeAllCookies(...)`
   
- `WebView.clearCache(true)`
   
- `WebStorage.getInstance().deleteAllData()`
   
- `WebView.clearFormData()`
   
- `WebView.clearHistory()`
   

И самое важное: даже если методы вызываются, данные могли остаться на диске. Поэтому нужно **проверить папку WebView** после закрытия приложения

--------

как чекать:



1. **Запустить приложение** и дойти до экрана с WebView (например, авторизация через Яндекс – `YandexAuthWebViewActivity`)
    
2. **Введи тестовые данные** или дай WebView загрузить страницу (она сохранит куки и кеш)
    
3. **Закрой приложение полностью** (смахни из Recents или выполни `adb shell am force-stop com.evgeniy.meetway`)
    
4. **Проверить, что осталось в папке WebView:**

   
    ```c
    adb shell run-as com.evgeniy.meetway ls -la /data/data/com.evgeniy.meetway/app_webview/
    adb shell run-as com.evgeniy.meetway cat /data/data/com.evgeniy.meetway/app_webview/Cookies 2>/dev/null
    ```
    

###  Как интерпретировать результат

- **Тест пройден**, если папка `app_webview` пуста или в ней нет следов чувствительных данных

- **Тест провален**, если там остались файлы с куками, кешем или localStorage, содержащие данные, которые ты вводил в WebView
-------

## Результаты теста 

**Приложение:** MeetWay Android
**Телефон:** OnePlus 8T (KB2003), Android 14, Magisk root

# 1 SAST: поиск WebView и методов очистки в коде

```bash
cd /Users/evgeniy/Documents/android-meetway

# Поиск WebView
grep -rn "WebView\|webView" --include="*.kt" app/src/main/

# Результат: YandexAuthWebViewActivity.kt – единственный WebView
```

**Настройки WebView (YandexAuthWebViewActivity.kt):**

| Параметр | Значение |
|--|--|
| `javaScriptEnabled` | `true` |
| `domStorageEnabled` | `true` |

**Методы очистки – grep по всему проекту:**

```bash
# Поиск методов очистки
grep -rn "CookieManager\|clearCache\|clearFormData\|clearHistory\|WebStorage\|removeAllCookies\|destroy()" --include="*.kt" app/src/main/

# Результат: НИ ОДИН МЕТОД НЕ НАЙДЕН
grep -rn "onDestroy" app/src/main/java/com/evgeniy/meetway/YandexAuthWebViewActivity.kt
# → onDestroy НЕ переопределён
```

# итого - тест провален! куки не чистятся !



| Метод | Найден | Где нужен |
|--|--|--|
| `CookieManager.removeAllCookies()` | ❌ Нет | `onDestroy()` |
| `WebView.clearCache(true)` | ❌ Нет | `onDestroy()` |
| `WebStorage.deleteAllData()` | ❌ Нет | `onDestroy()` |
| `WebView.clearFormData()` | ❌ Нет | `onDestroy()` |
| `WebView.clearHistory()` | ❌ Нет | `onDestroy()` |

# 2  DAST: запуск WebView на телефоне + проверка файлов

Перед запуском проверяю, есть ли уже данные WebView:

```bash
# Проверка до теста – app_webview не существует
adb shell su -c "ls -la /data/data/com.evgeniy.meetway/app_webview/ 2>/dev/null"
# → No such file or directory
```

Запускаю приложение:

```bash
adb shell monkey -p com.evgeniy.meetway -c android.intent.category.LAUNCHER 1
```

**👇 ДЖОНИ, НУЖНА ТВОЯ ПОМОЩЬ**

На телефоне открылось приложение MeetWay. Сделай:
1. Нажми **"Войти через Яндекс"** (или кнопку авторизации)
2. Если появится WebView со страницей Яндекса – просто **дождись загрузки страницы**
3. Потом **нажми назад** (или закрой экран) – вернись в приложение или на главный экран
4. **Смахни приложение из Recents** (список недавних приложений)

После этого напиши мне, что сделал.

**Результат DAST:**

Авторизация прошла **автоматически** – приложение сразу залогинилось, WebView (`YandexAuthWebViewActivity`) не открылся. Причина: Яндекс OAuth использует **внешний браузер** (Yandex Browser), в котором уже сохранена сессия. Custom Tabs / Browser – это отдельное приложение, его cookies не относятся к тесту.

```bash
# Проверка app_webview после авторизации
adb shell su -c "ls -la /data/data/com.evgeniy.meetway/app_webview/ 2>/dev/null"
# → Директории app_webview НЕ СУЩЕСТВУЕТ
# → Собственный WebView приложения НЕ ИСПОЛЬЗОВАЛСЯ
```

**Вывод:** При входе через Яндекс используется внешний браузер/Custom Tabs, а НЕ встроенный `YandexAuthWebViewActivity`. Это значит, что cookies и сессия управляются Яндекс Браузером – к приложению отношения не имеют.

Однако код `YandexAuthWebViewActivity` существует и может быть задействован при определённых условиях (например, если Яндекс Браузер не установлен или Custom Tabs недоступен). В этом случае данные НЕ будут очищены.

### Итог: FAIL ❌

| Проверка | Результат |
|--|--|
| Методы очистки в коде | ❌ НИ ОДНОГО |
| `onDestroy()` переопределён | ❌ Нет |
| `app_webview/` после авторизации | ⚠️ Не создавался (Custom Tabs) |
| Возможность утечки | Средняя – WebView может быть использован как fallback |

**Риск:** Если YandexAuthWebViewActivity будет использован (например, на устройстве без Яндекс Браузера), cookies, кэш и localStorage страницы `oauth.yandex.ru` останутся на диске после закрытия. Злоумышленник с ADB-доступом сможет их прочитать.

**Как исправить:**

Добавить очистку в `YandexAuthWebViewActivity.kt`:

```kotlin
override fun onDestroy() {
    super.onDestroy()
    
    // 1. Очистить cookies
    CookieManager.getInstance().removeAllCookies(null)
    CookieManager.getInstance().flush()
    
    // 2. Очистить кэш WebView (включая disk-кэш)
    webView.clearCache(true)
    
    // 3. Очистить Web Storage (localStorage + Web SQL)
    android.webkit.WebStorage.getInstance().deleteAllData()
    
    // 4. Очистить form data и историю
    webView.clearFormData()
    webView.clearHistory()
    
    // 5. Уничтожить WebView
    webView.destroy()
    
    // 6. Отключить DOM storage
    webView.settings.domStorageEnabled = false
}
```

Также переопределить `onPause()` с теми же вызовами - на случай, если Activity будет приостановлена без вызова `onDestroy()` (например, при повороте экрана)

---


