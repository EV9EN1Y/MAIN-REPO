# MASTG-TEST-0353: Runtime Use of Debugging Detection APIs
https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0353/

данный тест - продолжение статического теста 👉 [[R_SAST_обнаружениe_отладки-frida-lldb---итп+AI-LLM]]

------

предыдущий тест проверял _наличие_ в коде API для обнаружения отладки, то этот тест проверяет, вызываются ли они_на самом деле во время работы приложения

---------

тест проверяет, использует ли приложение API для обнаружения отладки в реальном времени. Статический анализ (предыдущий тест) мог показать, что код для проверки отладки _есть_, но он может быть "мертвым" (не вызываться) или быть обернут в условие, которое никогда не выполняется. **Динамический анализ** дает уверенность, что приложение действительно проверяет отладку в рантайме

Если проверки есть и они вызываются, приложение может предпринять защитные действия: выдать предупреждение, ограничить функционал или завершить работу. Если проверок нет или они не вызываются - приложение уязвимо для динамического анализа


---------

тест провожу на приложении  - [[0_MeetWay]]

андроид 14
буду использовать фриду 17.9 для хуков
попробую перехватить популярные методы, которые связаны с детектом отладки

сейчас на рутированном телефоне приложение при запуске - сразу вылетает!

-----


буду пробовать хукать методы , которыми приложение может пробовать детектить отладку!

------


скрипт который хукнет и отключит стандартные защитные методы 

```q

cat > /Users/evgeniy/Desktop/frida-android-scripts/hook_Debugging-Detection.js << 'EOF'




// Universal Anti-Debug & Detection Hook Script
console.log("[*] Universal Anti-Debug and Detection Bypass Script Started");

Java.perform(function() {
    // --- 1. Java/Kotlin Layer Hooks ---

    // 1.1. Основной метод проверки отладчика
    try {
        var DebugClass = Java.use("android.os.Debug");
        DebugClass.isDebuggerConnected.implementation = function() {
            console.log("[+] Java: Debug.isDebuggerConnected() called. Returning false.");
            return false;
        };
        console.log("[+] Hooked: Debug.isDebuggerConnected()");
    } catch(e) { console.log("[-] Could not hook Debug.isDebuggerConnected: " + e); }

    // 1.2. Проверка флага дебаггинга приложения (часто используется в защитах)
    try {
        var ApplicationInfo = Java.use("android.content.pm.ApplicationInfo");
        ApplicationInfo.flags.value = 0; // Сбрасываем флаг дебаггинга
        console.log("[+] Hooked: ApplicationInfo.flags (debuggable flag disabled)");
    } catch(e) { console.log("[-] Could not hook ApplicationInfo.flags: " + e); }

    // 1.3. Проверка системного свойства ro.debuggable
    try {
        var System = Java.use("java.lang.System");
        System.getProperty.overload('java.lang.String').implementation = function(key) {
            if (key === "ro.debuggable") {
                console.log("[+] Java: System.getProperty('ro.debuggable') called. Returning '0'.");
                return "0";
            }
            return this.getProperty(key);
        };
        console.log("[+] Hooked: System.getProperty('ro.debuggable')");
    } catch(e) { console.log("[-] Could not hook System.getProperty: " + e); }

    // 1.4. Блокировка попыток приложения самоуничтожиться
    try {
        var System = Java.use("java.lang.System");
        System.exit.implementation = function(code) {
            console.log("[+] Java: System.exit() called with code: " + code + ". Blocked.");
        };
        console.log("[+] Hooked: System.exit()");
    } catch(e) { console.log("[-] Could not hook System.exit: " + e); }

    try {
        var Process = Java.use("android.os.Process");
        Process.killProcess.implementation = function(pid) {
            console.log("[+] Java: Process.killProcess() called with pid: " + pid + ". Blocked.");
        };
        console.log("[+] Hooked: Process.killProcess()");
    } catch(e) { console.log("[-] Could not hook Process.killProcess: " + e); }

    // --- 2. Native Layer Hooks (libc.so) ---

    // 2.1. Блокировка ptrace() – системный вызов для трассировки процессов
    try {
        var ptracePtr = Module.findExportByName("libc.so", "ptrace");
        if (ptracePtr) {
            Interceptor.replace(ptracePtr, new NativeCallback(function (request, pid, addr, data) {
                console.log("[+] Native: ptrace() called. Request: " + request + ". Returning 0 (success).");
                return 0;
            }, 'int', ['int', 'int', 'pointer', 'pointer']));
            console.log("[+] Hooked: ptrace() in libc.so");
        }
    } catch(e) { console.log("[-] Could not hook ptrace: " + e); }

    // 2.2. Перехват fopen() для подмены файлов, которые читает приложение
    try {
        var fopenPtr = Module.findExportByName("libc.so", "fopen");
        if (fopenPtr) {
            Interceptor.attach(fopenPtr, {
                onEnter: function(args) {
                    var path = Memory.readUtf8String(args[0]);
                    if (path && path.indexOf("/proc/self/status") !== -1) {
                        console.log("[+] Native: Attempt to read /proc/self/status. Redirecting to fake.");
                        var fakePath = Memory.allocUtf8String("/data/local/tmp/fake_status");
                        args[0] = fakePath;
                        console.log("[+] Native: fopen() redirected path to fake file.");
                    }
                }
            });
            console.log("[+] Hooked: fopen() in libc.so (for /proc/self/status)");
        }
    } catch(e) { console.log("[-] Could not hook fopen: " + e); }

    // 2.3. Перехват open() как альтернатива fopen()
    try {
        var openPtr = Module.findExportByName("libc.so", "open");
        if (openPtr) {
            Interceptor.attach(openPtr, {
                onEnter: function(args) {
                    var path = Memory.readUtf8String(args[0]);
                    if (path && path.indexOf("/proc/self/status") !== -1) {
                        console.log("[+] Native: Attempt to open /proc/self/status. Redirecting to fake.");
                        var fakePath = Memory.allocUtf8String("/data/local/tmp/fake_status");
                        args[0] = fakePath;
                    }
                }
            });
            console.log("[+] Hooked: open() in libc.so");
        }
    } catch(e) { console.log("[-] Could not hook open: " + e); }

    // 2.4. Перехват strstr() для поиска подозрительных строк (например, 'frida')
    try {
        var strstrPtr = Module.findExportByName("libc.so", "strstr");
        if (strstrPtr) {
            Interceptor.attach(strstrPtr, {
                onEnter: function(args) {
                    var haystack = Memory.readUtf8String(args[0]);
                    this.isSuspicious = haystack && (haystack.indexOf("frida") !== -1 || haystack.indexOf("xposed") !== -1 || haystack.indexOf("magisk") !== -1);
                },
                onLeave: function(retval) {
                    if (this.isSuspicious) {
                        console.log("[+] Native: strstr() found suspicious string. Returning null.");
                        retval.replace(ptr(0));
                    }
                }
            });
            console.log("[+] Hooked: strstr() in libc.so (for Frida/Xposed/Magisk)");
        }
    } catch(e) { console.log("[-] Could not hook strstr: " + e); }

    // 2.5. Перехват abort() для предотвращения "панического" завершения
    try {
        var abortPtr = Module.findExportByName("libc.so", "abort");
        if (abortPtr) {
            Interceptor.attach(abortPtr, {
                onEnter: function(args) {
                    console.log("[+] Native: abort() called. Blocking to prevent crash.");
                    // Здесь можно добавить логику, чтобы продолжить выполнение
                }
            });
            console.log("[+] Hooked: abort() in libc.so");
        }
    } catch(e) { console.log("[-] Could not hook abort: " + e); }

    // 2.6. Перехват raise() для блокировки отправки сигналов
    try {
        var raisePtr = Module.findExportByName("libc.so", "raise");
        if (raisePtr) {
            Interceptor.attach(raisePtr, {
                onEnter: function(args) {
                    var signal = args[0].toInt32();
                    if (signal === 6) { // SIGABRT
                        console.log("[+] Native: raise(SIGABRT) called. Blocked.");
                    }
                }
            });
            console.log("[+] Hooked: raise() in libc.so");
        }
    } catch(e) { console.log("[-] Could not hook raise: " + e); }

    console.log("[*] All hooks are set. Device should appear clean to the app.");

    // Дополнительно: можно создать фейковый файл /data/local/tmp/fake_status
    // с TracerPid: 0 для обхода проверок, если они идут через системные вызовы.
});

console.log("[*] Universal Anti-Debug and Detection Bypass Script Loaded.");




EOF
```

запуск приложения скриптом 

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/frida-android-scripts/hook_Debugging-Detection.js
```


-----

###  как этот скрипт работает

1. **Блокировка Java/Kotlin проверок:**
    
    - **`Debug.isDebuggerConnected()`**: Перехватывает главный системный вызов, которым приложение проверяет, не подключен ли к нему отладчик. Скрипт всегда возвращает `false`, как будто отладчика нет (в идеале конечно, просто перехватывать методы, если есть, и смотреть их возвращаемое значение, и смотреть, чем можно подменить... И так далее итп ... )
      
    - **`ApplicationInfo.flags`**: Подменяет системный флаг, который показывает, что приложение запущено в режиме отладки (`FLAG_DEBUGGABLE`), делая его невидимым для проверок
      
    - **`System.getProperty('ro.debuggable')`**: Перехватывает чтение системного свойства `ro.debuggable` и возвращает `'0'` (отладка отключена)
      
    - **`System.exit()` и `Process.killProcess()`**: Не дает приложению самоуничтожиться, если оно обнаружит угрозу
      
2. **Блокировка нативных проверок (C/C++):**
    
    - **`ptrace()`**: Блокирует системный вызов, который используется для трассировки процессов. Если приложение пытается проверить, не трассируют ли его, скрипт имитирует успешный ответ, что трассировки нет
      
    - **`fopen()` и `open()`**: Перехватывает попытки приложения прочитать системный файл `/proc/self/status` (где хранится `TracerPid` – идентификатор отладчика) и перенаправляет его на специальный "фейковый" файл, где `TracerPid` равен `0`
      
    - **`strstr()`**: Блокирует поиск "запрещенных" строк в памяти приложения, таких как `frida`, `xposed`, `magisk`. Если приложение ищет эти слова, скрипт "не находит" их
      
    - **`abort()` и `raise()`**: Блокирует фатальные ошибки и сигналы, которые приложение может послать само себе, чтобы упасть при обнаружении угрозы


--------

запускаю фрида сервер на мобилке


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


```


-------

```js


evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/frida-android-scripts/hook_Debugging-Detection.js
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
Spawning `com.evgeniy.meetway`...
[*] Universal Anti-Debug and Detection Bypass Script Started
[*] Universal Anti-Debug and Detection Bypass Script Loaded.
Spawned `com.evgeniy.meetway`. Resuming main thread!
[KB2003::com.evgeniy.meetway ]-> [+] Hooked: Debug.isDebuggerConnected()
[-] Could not hook ApplicationInfo.flags: Error: Cannot access an instance field without an instance

[+] Hooked: System.getProperty('ro.debuggable')
[+] Hooked: System.exit()
[+] Hooked: Process.killProcess()

[-] Could not hook ptrace: TypeError: not a function
[-] Could not hook fopen: TypeError: not a function
[-] Could not hook open: TypeError: not a function
[-] Could not hook strstr: TypeError: not a function
[-] Could not hook abort: TypeError: not a function
[-] Could not hook raise: TypeError: not a function

[*] All hooks are set. Device should appear clean to the app.
[+] Java: Process.killProcess() called with pid: 9089. Blocked.
[+] Java: System.exit() called with code: 1. Blocked.

```

были перехвачены и залочены методы:

- **`System.exit()` и `Process.killProcess()`**: Не дает приложению самоуничтожиться, если оно обнаружит угрозу

- **`System.getProperty('ro.debuggable')`**: Перехватывает чтение системного свойства `ro.debuggable` и возвращает `'0'` (отладка отключена)


-------

со скриптом приложение запустилось!

## вывод!

защита от отладки и рута есть!
но она банальная!
не хвататет конечно же обфускации и запутывания логики!





