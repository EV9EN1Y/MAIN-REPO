 https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0264/
# MASTG-TEST-0264: Использование API StrictMode во время выполнения

тест провожу на приложении 👉 [[0_MeetWay]]

данный тест - это продолжение данного теста [[R_dast-check--StrictMode-LOGs]]

где нужно было проверить, не оставили ли разработчики включенным режим **StrictMode** в продакшн-версии приложения

**StrictMode** - это инструмент разработчика в Android, который помогает находить проблемы в коде, например, выполнение медленных операций ввода-вывода или сетевых запросов в главном потоке (UI-потоке). Когда StrictMode включен, приложение может логировать такие нарушения, показывать предупреждения или даже крашиться, чтобы разработчик заметил проблему

В продакшн-версии этот режим должен быть выключен, потому что:

1. Он создает лишнюю нагрузку на устройство, замедляя работу приложения
   
2. Логи StrictMode могут содержать детали реализации приложения (например, названия файлов, структуру базы данных, сетевые запросы). Злоумышленник, имеющий доступ к логам устройства (например, через ADB или сторонние приложения для чтения логов), может использовать эту информацию для понимания внутреннего устройства приложения и поиска уязвимостей
   

Тест проверяет, нет ли в продакшн-сборке логов StrictMode

--------
этот тест выполняется с помощью frida 
нужно пробовать перехватить методы 

- `StrictMode.setVmPolicy`
   
- `StrictMode.VmPolicy.Builder.penaltyLog`

--------


фрида уже у меня стоит на телефоне и на маке
как все настроено - можно прочесть в [[окруж+разблок+root+frida(на_android14)]]

тестирую по прежнему приложение [[0_MeetWay]]


------

```q
кидаю shell телефона

adb shell
su

запускаю сервер фриды на телефоне (терминал пк)

cd /data/local/tmp
chmod 755 frida-server-17.9.1-android-arm64
nohup ./frida-server-17.9.1-android-arm64 &

-------------

НА МАКЕ В НОВОМ ТЕМИНАЛЕ 

// кидаю порт
adb forward tcp:27042 tcp:27042

смотрю запущенные процессы

frida-ps -U

вижу PID своего приложение

10994   MeetWay
```


теперь захожу в консоль фриды

```q
frida -U -n MeetWay

я в консоли
```

сохраняю скрипт на пк себе в папку frida-android-scripts на десктопе

```js
cat > /Users/evgeniy/Desktop/frida-android-scripts/hook_strictmode.js << 'EOF'


Java.perform(function () {
    console.log("[*] Скрипт для перехвата StrictMode запущен. Ждем вызовов...");

    // 1. Перехват StrictMode.setVmPolicy
    try {
        var StrictMode = Java.use("android.os.StrictMode");
        StrictMode.setVmPolicy.implementation = function (policy) {
            console.log("[+] StrictMode.setVmPolicy() вызван!");
            console.log("[+] Policy: " + policy);
            console.log("[+] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.setVmPolicy(policy);
        };
        console.log("[+] Хук на StrictMode.setVmPolicy() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке StrictMode.setVmPolicy: " + e);
    }

    // 2. Перехват StrictMode.VmPolicy.Builder.penaltyLog()
    try {
        var Builder = Java.use("android.os.StrictMode$VmPolicy$Builder");
        Builder.penaltyLog.implementation = function () {
            console.log("[+] StrictMode.VmPolicy.Builder.penaltyLog() вызван!");
            console.log("[+] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.penaltyLog();
        };
        console.log("[+] Хук на StrictMode.VmPolicy.Builder.penaltyLog() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке penaltyLog: " + e);
    }

    // 3. Дополнительно: перехват StrictMode.setThreadPolicy
    try {
        var StrictMode = Java.use("android.os.StrictMode");
        StrictMode.setThreadPolicy.implementation = function (policy) {
            console.log("[+] StrictMode.setThreadPolicy() вызван!");
            console.log("[+] Policy: " + policy);
            console.log("[+] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.setThreadPolicy(policy);
        };
        console.log("[+] Хук на StrictMode.setThreadPolicy() установлен");
    } catch (e) {
        console.log("[-] Ошибка при хуке StrictMode.setThreadPolicy: " + e);
    }

    console.log("[*] Все хуки StrictMode установлены. Теперь взаимодействуйте с приложением.");
});


EOF
```


теперь запускаю этот скрипт в вместе с запуском приложения

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/frida-android-scripts/hook_strictmode.js
```

 теперь мне нужно взаимодействовать с моим приложением и в логах моего скрипта пытаться найти вот Strict методы

вот такой вывод я получаю: 
все  что здесь есть появилось только в момент запуска приложения, в процессе его работы хукнуть Strict методы не вышло, их нет, только при запуске есть это: 

```c
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/frida-android-scripts/hook_strictmode.js
     ____
    / _  |   Frida 17.9.1 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to KB2003 (id=6b62e119)
Spawned `com.evgeniy.meetway`. Resuming main thread!
[KB2003::com.evgeniy.meetway ]-> [*] Скрипт для перехвата StrictMode запущен. Ждем вызовов...
[+] Хук на StrictMode.setVmPolicy() установлен
[+] Хук на StrictMode.VmPolicy.Builder.penaltyLog() установлен
[+] Хук на StrictMode.setThreadPolicy() установлен
[*] Все хуки StrictMode установлены. Теперь взаимодействуйте с приложением.
[+] StrictMode.setThreadPolicy() вызван!
[+] Policy: [StrictMode.ThreadPolicy; mask=1107296260]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setThreadPolicy(Native Method)
	at android.os.StrictMode.initThreadDefaults(StrictMode.java:1519)
	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7401)
	at android.app.ActivityThread.handleBindApplication(Native Method)
	at android.app.ActivityThread.-$$Nest$mhandleBindApplication(ActivityThread.java:0)
	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2428)
	at android.os.Handler.dispatchMessage(Handler.java:106)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] StrictMode.VmPolicy.Builder.penaltyLog() вызван!
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode$VmPolicy$Builder.penaltyLog(Native Method)
	at android.os.StrictMode$VmPolicy$Builder.build(StrictMode.java:1232)
	at android.os.StrictMode.initVmDefaults(StrictMode.java:1557)
	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7402)
	at android.app.ActivityThread.handleBindApplication(Native Method)
	at android.app.ActivityThread.-$$Nest$mhandleBindApplication(ActivityThread.java:0)
	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2428)
	at android.os.Handler.dispatchMessage(Handler.java:106)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] StrictMode.setVmPolicy() вызван!
[+] Policy: [StrictMode.VmPolicy; mask=1082130464]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setVmPolicy(Native Method)
	at android.os.StrictMode.initVmDefaults(StrictMode.java:1557)
	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7402)
	at android.app.ActivityThread.handleBindApplication(Native Method)
	at android.app.ActivityThread.-$$Nest$mhandleBindApplication(ActivityThread.java:0)
	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2428)
	at android.os.Handler.dispatchMessage(Handler.java:106)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] StrictMode.setThreadPolicy() вызван!
[+] Policy: [StrictMode.ThreadPolicy; mask=1073741876]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setThreadPolicy(Native Method)
	at com.google.firebase.concurrent.CustomThreadFactory.lambda$newThread$0$com-google-firebase-concurrent-CustomThreadFactory(CustomThreadFactory.java:45)
	at com.google.firebase.concurrent.CustomThreadFactory$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
	at java.lang.Thread.run(Thread.java:1572)

[+] StrictMode.setThreadPolicy() вызван!
[+] Policy: [StrictMode.ThreadPolicy; mask=1073741876]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setThreadPolicy(Native Method)
	at com.google.firebase.concurrent.CustomThreadFactory.lambda$newThread$0$com-google-firebase-concurrent-CustomThreadFactory(CustomThreadFactory.java:45)
	at com.google.firebase.concurrent.CustomThreadFactory$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
	at java.lang.Thread.run(Thread.java:1572)

[+] StrictMode.setThreadPolicy() вызван!
[+] Policy: [StrictMode.ThreadPolicy; mask=1073741876]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setThreadPolicy(Native Method)
	at com.google.firebase.concurrent.CustomThreadFactory.lambda$newThread$0$com-google-firebase-concurrent-CustomThreadFactory(CustomThreadFactory.java:45)
	at com.google.firebase.concurrent.CustomThreadFactory$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
	at java.lang.Thread.run(Thread.java:1572)

[+] StrictMode.setThreadPolicy() вызван!
[+] Policy: [StrictMode.ThreadPolicy; mask=1073741876]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setThreadPolicy(Native Method)
	at com.google.firebase.concurrent.CustomThreadFactory.lambda$newThread$0$com-google-firebase-concurrent-CustomThreadFactory(CustomThreadFactory.java:45)
	at com.google.firebase.concurrent.CustomThreadFactory$$ExternalSyntheticLambda0.run(D8$$SyntheticClass:0)
	at java.lang.Thread.run(Thread.java:1572)

[+] StrictMode.setVmPolicy() вызван!
[+] Policy: [StrictMode.VmPolicy; mask=0]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setVmPolicy(Native Method)
	at androidx.compose.ui.platform.AndroidComposeView$Companion.addNotificationForSysPropsChange(AndroidComposeView.android.kt:2967)
	at androidx.compose.ui.platform.AndroidComposeView$Companion.access$addNotificationForSysPropsChange(AndroidComposeView.android.kt:2912)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2031)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] StrictMode.setVmPolicy() вызван!
[+] Policy: [StrictMode.VmPolicy; mask=1082130464]
[+] Stack trace:
java.lang.Exception
	at android.os.StrictMode.setVmPolicy(Native Method)
	at androidx.compose.ui.platform.AndroidComposeView$Companion.addNotificationForSysPropsChange(AndroidComposeView.android.kt:2976)
	at androidx.compose.ui.platform.AndroidComposeView$Companion.access$addNotificationForSysPropsChange(AndroidComposeView.android.kt:2912)
	at androidx.compose.ui.platform.AndroidComposeView.onAttachedToWindow(AndroidComposeView.android.kt:2031)
	at android.view.View.dispatchAttachedToWindow(View.java:22244)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3543)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3550)
	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:3389)
	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2765)
	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:10219)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1544)
	at android.view.Choreographer$CallbackRecord.run(Choreographer.java:1553)
	at android.view.Choreographer.doCallbacks(Choreographer.java:1109)
	at android.view.Choreographer.doFrame(Choreographer.java:994)
	at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:1527)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)


```


 че по итогу: 
 в ходе динамического анализа с использованием Frida были перехвачены вызовы API StrictMode:

1. `StrictMode.setThreadPolicy()` - вызван системой Android (`ActivityThread.handleBindApplication`) и библиотекой Firebase (`CustomThreadFactory`)
   
2. `StrictMode.VmPolicy.Builder.penaltyLog()` - вызван при инициализации системы
   
3. `StrictMode.setVmPolicy()` - вызван системой Android и библиотекой Jetpack Compose (`AndroidComposeView`)
   

Пример перехваченного вызова:
```q
 StrictMode.setThreadPolicy() вызван!
 
  Policy: [StrictMode.ThreadPolicy; mask=1107296260]
  
 Stack trace:
    at android.os.StrictMode.setThreadPolicy(Native Method)
    at android.os.StrictMode.initThreadDefaults(StrictMode.java:1519)
    at android.app.ActivityThread.handleBindApplication(ActivityThread.java:7401)
```

=============

Вызовы `StrictMode.setThreadPolicy()` и `StrictMode.setVmPolicy()`, которые происходят на этапе запуска приложения, когда система Android инициализирует его процессы .

Это явный признак того, что StrictMode был включен где-то в коде приложения

=============

вызовы `StrictMode.setThreadPolicy()` из `com.google.firebase.concurrent.CustomThreadFactory` Они означают, что Firebase внутри себя также применяет политики StrictMode

=============

Вызовы `StrictMode.setVmPolicy()` из `androidx.compose.ui.platform.AndroidComposeView` 

**почему это плохо:**  
Jetpack Compose - это современный фреймворк для создания UI. Когда StrictMode включен, Compose начнет логировать каждое нарушение в работе интерфейса: перерисовки, обращения к ресурсам, работу с памятью . Эти логи раскроют структуру твоего приложения: названия экранов, используемые данные, методы работы с состоянием

-----------

Все эти вызовы объединяет одно: они включают механизм логирования нарушений. Хотя сам факт вызова не гарантирует, что нарушение произошло, он создает потенциальную возможность для утечки информации. Логи StrictMode могут содержать названия файлов, структуру базы данных, сетевые запросы и другую критическую информацию . Более того, включенный StrictMode создает дополнительную нагрузку на процессор и память, замедляя работу приложения


ну и чтобы понять, насколько опасны находки - необходимо сомотреть сам код через JADX  |  + нужно смотреть другие тесты с проверкой логов динамически  [[R_dast-check--StrictMode-LOGs]]


------------------

**Вывод:**  
Приложение использует API StrictMode в рантайме, что противоречит требованиям безопасности для продакшн-версий

Использование StrictMode может приводить к генерации логов, раскрывающих детали реализации, и создавать дополнительную нагрузку. Тест **ПРОВАЛЕН**

**что нужно делать:**  
убедиться, что StrictMode включен только в debug-сборках (например, с помощью проверки `if (BuildConfig.DEBUG)`)
Для релизной версии все вызовы StrictMode должны быть удалены или закомментированы

