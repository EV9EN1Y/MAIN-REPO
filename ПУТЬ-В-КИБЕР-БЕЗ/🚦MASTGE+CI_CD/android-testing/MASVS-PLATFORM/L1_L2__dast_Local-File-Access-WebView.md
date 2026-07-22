# MASTG-TEST-0253: Runtime Use of Local File Access APIs in WebViews
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0253/


Тест проверяет, может ли злоумышленник через уязвимый WebView получить доступ к локальным файлам приложения и украсть их.

Уязвимость возникает при включении трех настроек:

1. **`setJavaScriptEnabled(true)`** - позволяет выполнять JavaScript.
   
2. **`setAllowFileAccess(true)`** (или не отключено явно) - разрешает WebView загружать локальные файлы.
   
3. **`setAllowFileAccessFromFileURLs(true)`** или **`setAllowUniversalAccessFromFileURLs(true)`** - разрешает JavaScript в локальных файлах читать другие локальные файлы.


Если эти настройки включены вместе, злоумышленник может внедрить вредоносный HTML-файл, который прочитает ваши локальные данные (пароли, ключи, базы данных) и отправит их на свой сервер

-----

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

скрипт

```q
cat > ~/Desktop/hook_webview6_full.js << 'EOF'

console.log("[*] Full WebView Hooking Script Loaded");

Java.perform(function() {
    // 1. Перехватываем создание WebView
    try {
        var WebView = Java.use("android.webkit.WebView");
        console.log("[+] Using android.webkit.WebView");

        // Хуким конструкторы
        var constructors = WebView.class.getDeclaredConstructors();
        for (var i = 0; i < constructors.length; i++) {
            var constructor = constructors[i];
            var paramTypes = constructor.getParameterTypes();
            var overloadArgs = [];
            for (var j = 0; j < paramTypes.length; j++) {
                overloadArgs.push(paramTypes[j].getName());
            }
            try {
                var overloadMethod = WebView[overloadArgs.join(',')];
                if (overloadMethod) {
                    overloadMethod.implementation = function() {
                        console.log("[🔍] WebView created");
                        var result = this[overloadArgs.join(',')].apply(this, arguments);
                        // Логируем настройки после создания
                        try {
                            var settings = this.getSettings();
                            if (settings) {
                                logAllSettings(settings);
                            }
                        } catch (e) { /* игнорируем */ }
                        return result;
                    };
                }
            } catch (e) { /* пропускаем недоступные конструкторы */ }
        }
    } catch (e) {
        console.log("[!] Error with android.webkit.WebView: " + e);
    }

    // 2. Перехватываем WebSettings (настройки)
    try {
        var WebSettings = Java.use("android.webkit.WebSettings");

        // Перехват setJavaScriptEnabled
        WebSettings.setJavaScriptEnabled.implementation = function(enabled) {
            console.log("[⚙️] setJavaScriptEnabled(" + enabled + ") called");
            var backtrace = getBacktrace();
            if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
            return this.setJavaScriptEnabled(enabled);
        };

        // Перехват setAllowFileAccess
        WebSettings.setAllowFileAccess.implementation = function(allow) {
            console.log("[⚙️] setAllowFileAccess(" + allow + ") called");
            var backtrace = getBacktrace();
            if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
            return this.setAllowFileAccess(allow);
        };

        // Перехват setAllowFileAccessFromFileURLs
        WebSettings.setAllowFileAccessFromFileURLs.implementation = function(allow) {
            console.log("[⚙️] setAllowFileAccessFromFileURLs(" + allow + ") called");
            var backtrace = getBacktrace();
            if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
            return this.setAllowFileAccessFromFileURLs(allow);
        };

        // Перехват setAllowUniversalAccessFromFileURLs
        WebSettings.setAllowUniversalAccessFromFileURLs.implementation = function(allow) {
            console.log("[⚙️] setAllowUniversalAccessFromFileURLs(" + allow + ") called");
            var backtrace = getBacktrace();
            if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
            return this.setAllowUniversalAccessFromFileURLs(allow);
        };

        // Перехват setAllowContentAccess (из теста 0251)
        WebSettings.setAllowContentAccess.implementation = function(allow) {
            console.log("[⚙️] setAllowContentAccess(" + allow + ") called");
            var backtrace = getBacktrace();
            if (backtrace) console.log("[📎] Backtrace:\n" + backtrace);
            return this.setAllowContentAccess(allow);
        };

        console.log("[*] All WebSettings hooks installed.");
    } catch (e) {
        console.log("[!] Error hooking WebSettings: " + e);
    }

    console.log("[*] Hooking complete. Ready to intercept WebView settings.");
});

// Функция для логирования всех настроек WebView
function logAllSettings(settings) {
    try {
        var js = settings.getJavaScriptEnabled();
        var file = settings.getAllowFileAccess();
        var fileFromUrls = settings.getAllowFileAccessFromFileURLs();
        var universal = settings.getAllowUniversalAccessFromFileURLs();
        var content = settings.getAllowContentAccess();

        console.log("[📊] WebView Current Settings:");
        console.log("   - JavaScriptEnabled: " + js);
        console.log("   - AllowFileAccess: " + file);
        console.log("   - AllowFileAccessFromFileURLs: " + fileFromUrls);
        console.log("   - AllowUniversalAccessFromFileURLs: " + universal);
        console.log("   - AllowContentAccess: " + content);

        // Проверка опасных комбинаций
        var dangerFile = (js && file && (fileFromUrls || universal));
        var dangerContent = (js && content && universal);

        if (dangerFile || dangerContent) {
            console.log("[⚠️] ОБНАРУЖЕНА ОПАСНАЯ КОМБИНАЦИЯ НАСТРОЕК!");
            if (dangerFile) console.log("[⚠️]   - Возможен доступ к локальным файлам (тест 0252/0253)");
            if (dangerContent) console.log("[⚠️]   - Возможен доступ к контент-провайдерам (тест 0250/0251)");
        } else {
            console.log("[✅] Опасные комбинации настроек не обнаружены.");
        }
    } catch (e) {
        console.log("[!] Error reading settings: " + e);
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
frida -U -l ~/Desktop/hook_webview6_full.js -f com.evgeniy.meetway
```


результат этого динамического теста анлогичен статич тесту




setJavaScriptEnabled - тру

setAllowFileAccess - нет в коде

setAllowFileAccessFromFileURLs - нет в коде

setAllowUniversalAccessFromFileURLs  - нет в коде

Поскольку два метода (`setAllowFileAccessFromFileURLs` и `setAllowUniversalAccessFromFileURLs`) **не используются**, то для современных версий Android (API >= 16) их значения по умолчанию – **`false`**. Это означает, что даже если `AllowFileAccess`включен, JavaScript не сможет читать другие локальные файлы

-------

**Результат:** Тест **MASTG-TEST-0252** пройден

**Обоснование:** В ходе статического анализа кода с помощью jadx было установлено, что в приложении используется WebView (`YandexAuthWebViewActivity`). При этом:

- Метод `setAllowFileAccessFromFileURLs` не используется, следовательно, применяется безопасное значение по умолчанию (`false`)
   
- Метод `setAllowUniversalAccessFromFileURLs` не используется, следовательно, применяется безопасное значение по умолчанию (`false`)
   
- Отсутствие этих настроек делает невозможным чтение локальных файлов из WebView даже при включенном `setJavaScriptEnabled(true)`
   

Таким образом, риск утечки локальных файлов через WebView отсутствует

--------

