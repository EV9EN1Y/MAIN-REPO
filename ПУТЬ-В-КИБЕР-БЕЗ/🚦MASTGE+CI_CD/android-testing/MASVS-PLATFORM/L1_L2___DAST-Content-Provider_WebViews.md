# MASTG-TEST-0251: Runtime Use of Content Provider Access APIs in WebViews
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0251/

тест **MASTG-TEST-0250** проверяет, не создает ли приложение **опасную комбинацию настроек WebView**, которая позволит злоумышленнику получить доступ к локальным файлам и данным через контент-провайдеры


Если в WebView одновременно включены три настройки:

1. **`setJavaScriptEnabled(true)`** - включен JavaScript (он нужен для работы многих функций)
   
2. **`setAllowContentAccess(true)` (или не отключен явно)** - разрешен доступ к контент-провайдерам через `content://` URI
   
3. **`setAllowUniversalAccessFromFileURLs(true)`** - разрешены кросс-доменные запросы из файлов
   

То злоумышленник, который сможет внедрить свой HTML/JS код в WebView (например, через XSS или загрузку вредоносной страницы), сможет обратиться к любому контент-провайдеру на устройстве и вытянуть чувствительные данные


-------

в данном тесте я динамически проверю работу данных методов


```q


cat > ~/Desktop/hook_webview.js << 'EOF'

// hook_webview.js – универсальный перехват WebView
// Запуск: frida -U -l ~/Desktop/hook_webview.js -f com.evgeniy.meetway

console.log("[*] WebView Hooking Script Loaded");

Java.perform(function() {
    // 1. Находим WebView класс (может быть android.webkit.WebView или androidx.webkit.WebView)
    var WebView = null;
    try {
        WebView = Java.use("android.webkit.WebView");
        console.log("[+] Using android.webkit.WebView");
    } catch (e) {
        try {
            WebView = Java.use("androidx.webkit.WebView");
            console.log("[+] Using androidx.webkit.WebView");
        } catch (e2) {
            console.log("[!] WebView class not found. Is the app using WebView?");
            return;
        }
    }

    // 2. Перехватываем конструкторы WebView (все варианты)
    var constructors = WebView.class.getDeclaredConstructors();
    for (var i = 0; i < constructors.length; i++) {
        var constructor = constructors[i];
        var paramTypes = constructor.getParameterTypes();
        var overloadArgs = [];
        for (var j = 0; j < paramTypes.length; j++) {
            overloadArgs.push(paramTypes[j].getName());
        }
        try {
            var overload = WebView.class.getDeclaredConstructor(paramTypes);
            // Хуким конструктор
            var overloadMethod = WebView[overloadArgs.join(',')];
            if (overloadMethod) {
                overloadMethod.implementation = function() {
                    console.log("[🔍] WebView constructor called with " + arguments.length + " args");
                    var result = this[overloadArgs.join(',')].apply(this, arguments);
                    // Проверяем настройки после создания
                    try {
                        var settings = this.getSettings();
                        if (settings) {
                            logWebViewSettings(settings);
                        }
                    } catch (e) {
                        console.log("[!] Error reading settings: " + e);
                    }
                    return result;
                };
            }
        } catch (e) {
            // Пропускаем конструкторы, которые нельзя перехватить
        }
    }

    // 3. Перехватываем setJavaScriptEnabled
    var WebSettings = Java.use("android.webkit.WebSettings");
    WebSettings.setJavaScriptEnabled.implementation = function(enabled) {
        console.log("[⚙️] setJavaScriptEnabled(" + enabled + ") called");
        var backtrace = getBacktrace();
        if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
        return this.setJavaScriptEnabled(enabled);
    };

    // 4. Перехватываем setAllowContentAccess
    WebSettings.setAllowContentAccess.implementation = function(allow) {
        console.log("[⚙️] setAllowContentAccess(" + allow + ") called");
        var backtrace = getBacktrace();
        if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
        return this.setAllowContentAccess(allow);
    };

    // 5. Перехватываем setAllowUniversalAccessFromFileURLs
    WebSettings.setAllowUniversalAccessFromFileURLs.implementation = function(allow) {
        console.log("[⚙️] setAllowUniversalAccessFromFileURLs(" + allow + ") called");
        var backtrace = getBacktrace();
        if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
        return this.setAllowUniversalAccessFromFileURLs(allow);
    };

    console.log("[*] All hooks installed.");
});

// Функция для логирования настроек WebView
function logWebViewSettings(settings) {
    try {
        var jsEnabled = settings.getJavaScriptEnabled();
        var contentAccess = settings.getAllowContentAccess();
        var universalAccess = settings.getAllowUniversalAccessFromFileURLs();
        
        console.log("[📊] WebView Settings:");
        console.log("   - JavaScriptEnabled: " + jsEnabled);
        console.log("   - AllowContentAccess: " + contentAccess);
        console.log("   - AllowUniversalAccessFromFileURLs: " + universalAccess);
        
        if (jsEnabled && contentAccess && universalAccess) {
            console.log("[⚠️] КРИТИЧЕСКАЯ КОМБИНАЦИЯ: все три настройки включены!");
        } else {
            console.log("[✅] Опасная комбинация настроек не обнаружена.");
        }
    } catch (e) {
        console.log("[!] Error reading WebView settings: " + e);
    }
}

// Функция для получения стека вызовов
function getBacktrace() {
    try {
        var Exception = Java.use("java.lang.Exception");
        var exception = Exception.$new();
        var log = Java.use("android.util.Log");
        return log.getStackTraceString(exception);
    } catch (e) {
        return null;
    }
}

EOF
```

```q
frida -U -l ~/Desktop/hook_webview.js -f com.evgeniy.meetway
```

если все три метода стоят в тру - тест провален!

--------

запускаю

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


setJavaScriptEnabled - тру

setAllowContentAccess -  false

setAllowUniversalAccessFromFileURLs -  не был найден

-------

**Результат:** Тест **MASTG-TEST-0250** пройден

**Обоснование:** В ходе статического анализа кода с помощью jadx было установлено, что в приложении используется WebView. При этом:

- Метод `setAllowContentAccess` явно установлен в `false`, что блокирует доступ к контент-провайдерам из WebView
   
- Метод `setAllowUniversalAccessFromFileURLs` не используется, следовательно, применяется безопасное значение по умолчанию (`false`)
   

Таким образом, критическая комбинация настроек, позволяющая злоумышленнику через WebView получить доступ к локальным данным через контент-провайдеры, отсутствует