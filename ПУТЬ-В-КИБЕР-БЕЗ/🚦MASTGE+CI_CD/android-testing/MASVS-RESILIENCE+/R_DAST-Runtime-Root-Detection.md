https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0325/#overview
# MASTG-TEST-0325: Runtime Use of Root Detection Techniques

-------

тест провожу на приложении 👉 [[0_MeetWay]]


этот же тест но статически здесь = [[R_SAST-Root-Detection+ai-LLM]]



тест **MASTG-TEST-0324**,  проверяет, есть ли в приложении **механизмы обнаружения рута (root detection)**

Задача этого теста - найти в коде приложения следы того, что оно пытается понять, не взломано ли устройство, на котором оно запущено

Тест проверяет, внедрили ли разработчики в приложение защиту от работы на рутованных (взломанных) устройствах. Это делается в данном тесте с помощью **ДИНАМИЧЕСКОГО анализа** при работе приложения. В идеале, приложение должно проверять, не нарушена ли целостность системы, и если да - ограничивать свой функционал или совсем не запускаться, чтобы защитить данные пользователя

Тест ищет в коде:

- Проверки на наличие характерных для рута файлов (например, `su`).
    
- Вызовы API, которые могут сигнализировать о руте.
    
- Использование сторонних библиотек для обнаружения рут


--------


### типичные признаки root detection

при нахождении признаков - вызовов методов, проверяющих наличие этих файлов или свойств, значит, root detection реализован, нужно детально проверить - есть ли такая защита или нет


пейлоад нагрузка для поиска 


```
- `su` (поиск проверок на наличие файла `su`)
    
- `Superuser.apk`
    
- `Magisk`
    
- `test-keys`
    
- `isDeviceRooted`
    
- `root`
    
- `Build.TAGS` (проверка на `test-keys`)
    
- `Build.MANUFACTURER` (может использоваться для проверки на эмулятор/рут)
    
- `isRooted` или `isDeviceRooted` (часто встречаются в библиотеках обнаружения рута)
    
- `RootBeer` (название популярной библиотеки для обнаружения рута)
    
- `which su` (проверка доступа к команде `su`)
```


-------

в тесте [[R_SAST-Root-Detection+ai-LLM]] было обнаружено, что детект рут защита
 В коде реализована, но не включена.

### План данного теста

 проведу динамический тест с отключённой защитой и проверим реакцию

Далее, я активирую детекцию root в приложении, переустановлю его, и проведу заново тот же самый тест, который в этот раз должен показать, что защита сработала

Я сделаю так что приложение вылетит при обнаружении взлома,

Следующий мой шаг это я попытаюсь полностью обойти установленную мною защиту

--------

## провожу динамический тест на детект рута

тест провожу на андроид 14
телефон с рутом
фрида уже установленна и приложение meetway тоже стоит (функции детекции рута там отключены мной к коде)

запускаю шел и фрида сервер на телефоне

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

12435   MeetWay
```


## создаю срипт для фрида который будет детектировать рут

```js
cat > /Users/evgeniy/Desktop/hook_and_spawn_root.js << 'EOF'




// hook_and_spawn_root.js
Java.perform(function() {
    console.log("[*] === Root Detection Hook Script Started ===");

    // --- 1. Основной класс SecurityDetector (ваша находка) ---
    var SecurityDetector = null;
    try {
        SecurityDetector = Java.use("com.evgeniy.meetway.util.SecurityDetector");
        console.log("[+] Found: SecurityDetector class");

        // Перехват основного метода
        SecurityDetector.isDeviceCompromised.implementation = function(context) {
            console.log("\n[!] ===== SecurityDetector.isDeviceCompromised() CALLED =====");
            var result = this.isDeviceCompromised(context);
            console.log("[!] Result: " + result);
            // Выводим стек вызовов, чтобы понять, откуда идет вызов
            console.log("[!] Stack trace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return result;
        };

        // Перехват всех внутренних методов проверок
        var methodsToHook = [
            "checkRootBinaries",
            "checkWritableSystemPartitions",
            "checkDangerousPackages",
            "checkHooks",
            "checkCustomRecovery"
        ];

        methodsToHook.forEach(function(methodName) {
            try {
                SecurityDetector[methodName].implementation = function() {
                    console.log("[+] SecurityDetector." + methodName + "() called");
                    // Для методов с параметрами и без
                    var result = (arguments.length > 0) ? this[methodName].apply(this, arguments) : this[methodName]();
                    console.log("[+] Result of " + methodName + ": " + result);
                    return result;
                };
                console.log("[+] Hooked: " + methodName);
            } catch (e) {
                console.log("[-] Could not hook " + methodName + ": " + e);
            }
        });

    } catch (e) {
        console.log("[-] SecurityDetector class not found: " + e);
    }

    // --- 2. Перехват системных API, часто используемых для проверки рута ---
    // a) Проверка наличия файла su
    try {
        var File = Java.use("java.io.File");
        File.exists.implementation = function() {
            var path = this.getPath();
            var result = this.exists();
            if (path.indexOf("su") !== -1 || path.indexOf("frida") !== -1 || path.indexOf("magisk") !== -1) {
                console.log("[+] File.exists() called for suspicious path: " + path + " -> " + result);
            }
            return result;
        };
        console.log("[+] Hooked: java.io.File.exists() for suspicious paths");
    } catch (e) {
        console.log("[-] Error hooking File.exists: " + e);
    }

    // b) Проверка через PackageManager (поиск root-приложений)
    try {
        var PackageManager = Java.use("android.content.pm.PackageManager");
        PackageManager.getPackageInfo.overload('java.lang.String', 'int').implementation = function(packageName, flags) {
            var result = this.getPackageInfo(packageName, flags);
            var suspiciousPackages = ["com.topjohnwu.magisk", "eu.chainfire.supersu", "com.noshufou.android.su"];
            if (suspiciousPackages.indexOf(packageName) !== -1) {
                console.log("[+] PackageManager.getPackageInfo() called for suspicious package: " + packageName);
            }
            return result;
        };
        console.log("[+] Hooked: PackageManager.getPackageInfo()");
    } catch (e) {
        console.log("[-] Error hooking PackageManager: " + e);
    }

    // c) Перехват Runtime.exec() (выполнение shell-команд)
    try {
        var Runtime = Java.use("java.lang.Runtime");
        Runtime.exec.overload('java.lang.String').implementation = function(command) {
            console.log("[+] Runtime.exec() called with command: " + command);
            // Проверяем, не пытаются ли запускать команду 'su' или 'which su'
            if (command.indexOf("su") !== -1 || command.indexOf("which") !== -1) {
                console.log("[!] Potentially root-related command executed: " + command);
            }
            return this.exec(command);
        };
        console.log("[+] Hooked: Runtime.exec()");
    } catch (e) {
        console.log("[-] Error hooking Runtime.exec: " + e);
    }

    // d) Перехват проверки System.getProperty (например, ro.debuggable)
    try {
        var System = Java.use("java.lang.System");
        System.getProperty.overload('java.lang.String').implementation = function(key) {
            var value = this.getProperty(key);
            if (key.indexOf("ro.") !== -1 || key.indexOf("debuggable") !== -1) {
                console.log("[+] System.getProperty() called for key: " + key + " -> value: " + value);
            }
            return value;
        };
        console.log("[+] Hooked: System.getProperty()");
    } catch (e) {
        console.log("[-] Error hooking System.getProperty: " + e);
    }

    console.log("[*] === All hooks are set. Spawning app and waiting for events... ===");
});




EOF
```



запустил скрипт

```c
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_and_spawn_root.js
```

результаты запуска

```q
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_and_spawn_root.js
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
[KB2003::com.evgeniy.meetway ]-> [*] === Root Detection Hook Script Started ===
[+] Found: SecurityDetector class
[+] Hooked: checkRootBinaries
[+] Hooked: checkWritableSystemPartitions
[+] Hooked: checkDangerousPackages
[+] Hooked: checkHooks
[+] Hooked: checkCustomRecovery
[+] Hooked: java.io.File.exists() for suspicious paths
[+] Hooked: PackageManager.getPackageInfo()
[+] Hooked: Runtime.exec()
[+] Hooked: System.getProperty()
[*] === All hooks are set. Spawning app and waiting for events... ===
[KB2003::com.evgeniy.meetway ]->
```

приложение запустилось ,  все хуки были успешно установлены - приложение потыкал
скрипт фриды молчит, приложение запустилось спокойно на рутированном телефоне 

на данный момент - тест ПРОВАЛЕН ❌ (ТАК как и фрида не нашла динамической защиты, и само приложение без проблем работает на рут телефоне с Magisk)

-------------


### теперь я активирую детекцию root в приложении, переустановлю его, и проведу заново тот же самый тест, который в этот раз должен показать, что защита сработала

открываю андроид студию - восстанавливаю работу рут детекции

за нее там отвечает класс SecurityDetector
в нем следующие методы
```
┌───┬────────────────────┬─────────────────────────────────────────────────────┐
│ № │ Метод              │ Что проверяет                                       │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 1 │ checkRootViaExec() │ su -c id — пытается реально выполнить su. Magisk    │
│   │                    │ перехватывает. Если вернулся uid=0 — root есть.     │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 2 │ checkMagiskFiles() │ Файлы Magisk в /data/adb/magisk/,                   │
│   │                    │ /data/adb/modules/ — они есть на диске              │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 3 │ checkMagiskMount() │ Читает /proc/1/mounts — видит Magisk tmpfs          │
│   │                    │ монтирования                                        │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 4 │ checkTestKeys()    │ Build.TAGS — если test-keys, прошивка кастомная     │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 5 │ checkSelinux()     │ Читает /sys/fs/selinux/enforce + getenforce         │
├───┼────────────────────┼─────────────────────────────────────────────────────┤
│ 6 │ checkFridaPort()   │ Пытается открыть socket на 127.0.0.1:27042 — порт   │
│   │                    │ Frida                                               │
└───┴────────────────────┴─────────────────────────────────────────────────────┘

```

1. В onCreate() — строка 87:

```kotlin
  // Было: закомментировано
  // detectCompromisedDevice()
  // Log.i(TAG, "✅ Детекция рута пропущена (отладка)")

  // Стало: активно
  detectCompromisedDevice()
```

2. В detectCompromisedDevice() — строка 153:

```kotlin
  // Было: без контекста
  // SecurityDetector.isDeviceCompromised()

  // Стало: с контекстом
  SecurityDetector.isDeviceCompromised(this)
```

Теперь при запуске приложения:
1. Вызывается SecurityDetector.isDeviceCompromised(this)
2. Проверяются: su бинарники, запись в /system, Magisk/SuperSU/Xposed,
   Frida-server, TWRP
3. Если хоть что-то найдено - killProcess + System.exit(1) - приложение падает



переустановил приложение

запускаю приложение

приложение зависает на запуске теперь!
и вижу свои логи о взломе

  ⚠️⚠️⚠️ Обнаружено взломанное устройство! Сработали проверки: exec_su
2026-06-30 01:53:26.303  8978-8978  MeetWayApp              com.evgeniy.meetway                  

⚠️⚠️⚠️ ОБНАРУЖЕНО ВЗЛОМАННОЕ УСТРОЙСТВО!
2026-06-30 01:53:26.303  8978-8978  System.err              com.evgeniy.meetway                  W 

⚠️ Внимание! Обнаружен root/jailbreak. Приложение не может быть запущено.
2026-06-30 01:53:26.303  8978-8978  Process                 com.evgeniy.meetway                  I  Sending signal. PID: 8978 SIG: 9


отлично! рут детекекция работает как надо и дает запустить приложение, так как определяет что телефон взломан!! ОТЛИЧНО!!


теперь запущу скрипт фрида, который попробует хукнуть динамически данные функции


запускаю 

```q
кидаю shell телефона

adb shell
su


уберу старые процессы (если нужно)
pkill -9 frida
ps -e | grep frida

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

12435   MeetWay
```

теперь запуск скрипта

```c
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_and_spawn_root.js
```

результат запуска скрипта:

```q
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_and_spawn_root.js
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
[KB2003::com.evgeniy.meetway ]-> [*] === Root Detection Hook Script Started ===
[+] Found: SecurityDetector class
[-] Could not hook checkRootBinaries: TypeError: cannot set property 'implementation' of undefined
[-] Could not hook checkWritableSystemPartitions: TypeError: cannot set property 'implementation' of undefined
[-] Could not hook checkDangerousPackages: TypeError: cannot set property 'implementation' of undefined
[-] Could not hook checkHooks: TypeError: cannot set property 'implementation' of undefined
[-] Could not hook checkCustomRecovery: TypeError: cannot set property 'implementation' of undefined

[+] Hooked: java.io.File.exists() for suspicious paths
[+] Hooked: PackageManager.getPackageInfo()
[+] Hooked: Runtime.exec()
[+] Hooked: System.getProperty()
[*] === All hooks are set. Spawning app and waiting for events... ===

[!] ===== SecurityDetector.isDeviceCompromised() CALLED =====
[+] File.exists() called for suspicious path: /data/adb/magisk/magisk.db -> false
[+] File.exists() called for suspicious path: /data/adb/magisk.db -> false
[+] File.exists() called for suspicious path: /data/adb/magisk/util_functions.sh -> false
[+] File.exists() called for suspicious path: /data/adb/.overlayfs_supported -> false
[+] Runtime.exec() called with command: cat /proc/1/mounts

[!] Result: true

Process terminated
[KB2003::com.evgeniy.meetway ]->

Thank you for using Frida!
```

скрипт отработал, перехватил все, что нужно, и показал, что приложение активно проверяет систему на наличие рута!!!!

отлично!!

**Результат запуска динамического теста:**  
В ходе динамического анализа с использованием Frida были перехвачены и подтверждены вызовы методов класса `com.evgeniy.meetway.util.SecurityDetector`, отвечающих за обнаружение рута

- Зафиксирован вызов основного метода `SecurityDetector.isDeviceCompromised()`, вернувший значение `true`
   
- Перехвачены проверки на наличие файлов, связанных с Magisk (например, `magisk.db`).
   
- Перехвачен запуск команды `cat /proc/1/mounts` для анализа смонтированных разделов
   
- После завершения проверок приложение корректно завершило свою работу (`Process terminated`), что свидетельствует о срабатывании механизма защиты
   

#### вывод:  
Приложение активно использует механизмы обнаружения рута во время выполнения, что подтверждает наличие активной защиты от работы на скомпрометированных устройствах. 
Тест **MASTG-TEST-0325** считается **ПРОЙДЕННЫМ**


этот же тест но статически здесь = [[R_SAST-Root-Detection+ai-LLM]]

---------

# обход защиты детекции рута!!


план - перехватить методы с помощью Фрида, которые отвечают за детекцию взломанного телефона

Подменить значение методах на противоположные, тем самым обойти защиту детекцией взлома, чтобы приложение полноценно запустилось!

****
скрипт перехвата функций и подмены значений
```q
cat > /Users/evgeniy/Desktop/bypass_root_fixed.js << 'EOF'






Java.perform(function() {
    console.log("[*] === Bypass Root Detection (FIXED) ===");

    // ─── 1. Перехват SecurityDetector.isDeviceCompromised() ───
    // Это главный вход. Если вернуть false — все внутренние проверки
    // (checkRootViaExec, checkMagiskFiles, checkFridaPort, и т.д.)
    // НИКОГДА не будут вызваны.
    try {
        var SecurityDetector = Java.use("com.evgeniy.meetway.util.SecurityDetector");
        SecurityDetector.isDeviceCompromised.implementation = function(context) {
            console.log("[!] isDeviceCompromised() called — FORCING return false");
            return false;
        };
        console.log("[+] SecurityDetector.isDeviceCompromised() → FORCED false");
    } catch (e) {
        console.log("[-] Cannot hook SecurityDetector: " + e);
    }

    // ─── 2. Runtime.exec() — НЕ БЛОКИРУЕМ ───
    // Просто логируем, чтобы видеть что вызывается.
    // НЕ возвращаем null — это ломает приложение!
    try {
        var Runtime = Java.use("java.lang.Runtime");
        Runtime.exec.overload('java.lang.String').implementation = function(command) {
            console.log("[LOG] exec: " + command.substring(0, 120));
            return this.exec(command);
        };
        console.log("[+] Runtime.exec() — только лог, без блокировки");
    } catch (e) {
        console.log("[-] Cannot hook Runtime.exec: " + e);
    }

    // ─── 3. File.exists() — только лог ───
    try {
        var File = Java.use("java.io.File");
        File.exists.implementation = function() {
            var path = this.getPath();
            var result = this.exists();
            if (path.indexOf("su") !== -1 || path.indexOf("magisk") !== -1 ||
                path.indexOf("frida") !== -1 || path.indexOf("adb") !== -1) {
                console.log("[LOG] File.exists(" + path + ") → " + result);
            }
            return result;
        };
        console.log("[+] File.exists() — только лог");
    } catch (e) {
        console.log("[-] Cannot hook File.exists: " + e);
    }

    console.log("[*] === Bypass ready. App should launch normally. ===");
});




EOF
```

запуск приложения со скриптом обхода!

```c
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/bypass_root_fixed.js
```

результат - приложение  запустилось  И ПОЛНОСТЬЮ РАБОТАЕТ!!! УРА!!!

удалось обойти защиту!

```js
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/bypass_root_fixed.js
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
Spawning `com.evgeniy.meetway`...                               Spawned `com.evgeniy.meetway`. Resuming main thread!
[KB2003::com.evgeniy.meetway ]-> [*] === Bypass Root Detection (FIXED) ===
[+] SecurityDetector.isDeviceCompromised() → FORCED false
[+] Runtime.exec() — только лог, без блокировки
[+] File.exists() — только лог
[*] === Bypass ready. App should launch normally. ===
[!] isDeviceCompromised() called — FORCING return false
Device lost
[KB2003::com.evgeniy.meetway ]->

Thank you for using Frida!
evgeniy@Evgeniys-MacBook-Pro-2 ~ %
```




--------

### как работает обход:

1. SAST в JADX и нашел класс `SecurityDetector`. Внутри него был метод `isDeviceCompromised()`, который объединял все проверки. Это был главный "переключатель", который решал, взломано устройство или нет
   
2. **Динамический анализ (MASTG-TEST-0325):**  написал скрипт, который перехватывает именно `isDeviceCompromised()`. Когда он сработал, в логах появилось `[!] ===== SecurityDetector.isDeviceCompromised() CALLED =====`
3. Это подтвердило, что приложение действительно вызывает этот метод в рантайме
4. ну а дальше подменил true на false

--------

## вывод!

тест пройден успешно

защита есть!

динамически защита работает!!

это очень хорошо!

но защита легко обходится, так как нет обфускации методов и запутывания логики!

чтобы сделать все более правильно - необходимо добавить обфускацию и запутывание методов детекции рута!

