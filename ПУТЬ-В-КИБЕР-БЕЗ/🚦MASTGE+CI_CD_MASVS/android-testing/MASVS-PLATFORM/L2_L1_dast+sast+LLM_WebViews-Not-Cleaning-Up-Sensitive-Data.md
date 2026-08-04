# MASTG-TEST-0320: WebViews Not Cleaning Up Sensitive Data
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0320/



тест проверяет, не остаются ли в WebView следы чувствительных данных (пароли, куки, локальное хранилище) после того, как ты закрыл приложение. Если данные не удаляются, кто-то с доступом к файловой системе телефона (или через бэкап) может их вытащить.

###  Суть теста MASTG-TEST-0320

WebView, как и браузер, может сохранять у себя:

- **Cookies** - сессионные данные, токены
    
- **Кеш** =- локальные копии страниц и ресурсов
    
- **localStorage / Web SQL** - данные JavaScript (могут содержать PII)
    
- **AppCache** - устаревший, но всё ещё возможный способ хранения
    

Приложение должно очищать* эти данные, когда они больше не нужны, особенно после выхода из аккаунта или закрытия экрана с WebView. Тест проверяет, вызываются ли соответствующие методы очистки:

- `CookieManager.getInstance().removeAllCookies(...)`
   
- `WebView.clearCache(true)`
   
- `WebStorage.getInstance().deleteAllData()`
   
- `WebView.clearFormData()`
   
- `WebView.clearHistory()`
   

И самое важное: даже если методы вызываются, данные могли остаться на диске. Поэтому нужно **проверить папку WebView** после закрытия приложения

---------


фрида везде стоит уже

```q
кидаю shell телефона

adb shell
su

запускаю сервер фриды на телефоне

cd /data/local/tmp
chmod 755 frida-server-17.9.1-android-arm64
nohup ./frida-server-17.9.1-android-arm64 &


НА МАКЕ В НОВОМ ТЕМИНАЛЕ 

// кидаю порт
adb forward tcp:27042 tcp:27042

смотрю запущенные процессы

frida-ps -U

вижу PID своего приложение

17576   MeetWay


гоу!!

frida -U -l ~/Desktop/hook_webview.js -f com.evgeniy.meetway
```

## и сразу запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```


------



**Шаг 1 - SAST: поиск WebView и методов очистки в коде**

**Где используется WebView:**

| Файл | Назначение |
|--|--|
| `YandexAuthWebViewActivity.kt` | Yandex OAuth – загружает страницу `https://oauth.yandex.ru/authorize`, перехватывает токен через `onPageStarted` |

**Настройки WebView:**

| Параметр | Значение | Риск |
|--|--|--|
| `javaScriptEnabled` | `true` | ✅ Необходимо для OAuth-формы |
| `domStorageEnabled` | `true` | ⚠️ Включает localStorage – данные JS могут сохраняться между сессиями |

**Поиск методов очистки во всём проекте:**

| Метод | Найден | Где должен быть |
|--|--|--|
| `CookieManager.removeAllCookies()` | ❌ **Нет** | `onDestroy()` / `onPause()` |
| `WebView.clearCache(true)` | ❌ **Нет** | `onDestroy()` |
| `WebStorage.getInstance().deleteAllData()` | ❌ **Нет** | `onDestroy()` |
| `WebView.clearFormData()` | ❌ **Нет** | `onDestroy()` |
| `WebView.clearHistory()` | ❌ **Нет** | `onDestroy()` |

**Методы жизненного цикла `YandexAuthWebViewActivity`:**
- `onCreate()` - ✅ есть, создаёт WebView и грузит URL
- `handleTokenFromUrl()` - ✅ есть, парсит токен
- `finishWithError()` - ✅ есть, завершает с ошибкой
- `onDestroy()` - ❌ **НЕ переопределён** - никакой очистки WebView не происходит
- `onPause()` / `onResume()` - ❌ Не переопределены


-------


**Шаг 2 - DAST: запуск на телефоне + проверка файловой системы**

Запускаю приложение на OnePlus 8T (USB):

```bash
# Запуск MeetWay
adb shell monkey -p com.evgeniy.meetway -c android.intent.category.LAUNCHER 1 2>/dev/null

# Проверка PID – приложение запущено
adb shell pidof com.evgeniy.meetway
# → 25754

# Попытка открыть YandexAuthWebViewActivity (единственный WebView в приложении)
adb shell am start -n com.evgeniy.meetway/.YandexAuthWebViewActivity
# → Permission Denial: not exported from uid 10339
# Activity не экспортирована (exported=false в манифесте) –
# открыть её можно только изнутри самого приложения

# Проверка файловой системы приложения на WebView-данные
adb shell su -c "ls -la /data/data/com.evgeniy.meetway/app_webview/ 2>/dev/null"
# → No such file or directory (WebView не инициализирован)

adb shell su -c "ls -la /data/data/com.evgeniy.meetway/cache/"
# → cache/image_cache (только кэш картинок, не WebView)

adb shell su -c "ls -la /data/data/com.evgeniy.meetway/databases/"
# → firestore.%5BDEFAULT%5D... (только Firestore, не Web Storage)
```

**Итоги динамической проверки:**
- YandexAuthWebViewActivity не стартует напрямую, но это не влияет на вердикт
- WebView не хранит данные в `/app_webview/` на момент проверки (ещё не использовался)
- После закрытия WebView данные **не будут очищены** - кода очистки нет




----------


**Итог: ❌❌❌ ПРОВАЛЕН ТЕСТ  ❌❌❌**

Приложение использует WebView для Yandex OAuth, но **не очищает**:
- Cookies (токены сессии Яндекса)
- Кэш WebView (страница авторизации)
- localStorage (если Яндекс сохраняет данные JS)
- Form data (если пользователь вводил данные на странице)
- History

**Критичность: Средняя** - WebView загружает только страницу Yandex OAuth (oauth.yandex.ru/authorize). После успешной авторизации токен извлекается и сохраняется в SecureStorage, а WebView-activity закрывается. Но cookies и кэш страницы логина остаются на диске

Сценарий атаки: злоумышленник с доступом к файловой системе телефона (USB-debugging, бэкап, вредоносное приложение) может прочитать cookies сессии Yandex из `/data/data/com.evgeniy.meetway/app_webview/Cookies` и получить доступ к аккаунту пользователя






**Как исправить:**

Добавить очистку в `YandexAuthWebViewActivity`:
```kotlin
override fun onDestroy() {
    super.onDestroy()
    
    // 1. Очистить cookies
    CookieManager.getInstance().removeAllCookies(null)
    CookieManager.getInstance().flush()
    
    // 2. Очистить кэш WebView
    webView.clearCache(true)
    
    // 3. Очистить Web Storage (localStorage + Web SQL)
    WebStorage.getInstance().deleteAllData()
    
    // 4. Очистить form data и историю
    webView.clearFormData()
    webView.clearHistory()
    
    // 5. Удалить WebView, чтобы освободить память
    webView.destroy()
    
    // 6. Очистить DOM storage
    webView.settings.domStorageEnabled = false
}
```

ИЛИ переопределить `onPause()` с теми же вызовами (для подстраховки, если Activity будет убита без `onDestroy`).

---


