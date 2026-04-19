**Shadow - это твик (специальная программа) для iPhone с джейлбрейком, главная задача которого - обмануть приложения и заставить их думать, что телефон не взломан. Простыми словами, он прячет сам факт наличия джейлбрейка от проверок приложений

-----

## 🟡 пытаюсь обойти защиту от джейлбрейка в приложении DVIA-v2 Shadow

запускаю приложение - оно сворачивается само

при этом, я через фрида не вижу процесс этого приложения, хотя оно висит во бекграунде, но при попытке открыть - сворачивается

не получается увидеть процесс самого приложения

--------
`frida-ps -Uai` полное название приложения
     DVIA-v2       com.highaltitudehacks.DVIAswiftv2

----

попробую через твик Shadow   (сразу скажу, что не помогло)
качаю это 
https://ios.jjolano.me
и это
https://opa334.github.io/

и устанавливаю:
- `altlist`
- - `libsandy`
- - `HookKit Framework`
- `RootBridge Framework`
- - `Shadow`
- (часть автоматом ставится)

теперь, после того, как Shadow был установлен
идем в настройки айона
в самый низ - там Shadow

открыл Shadow
далее - приложения - DVIA-v2 

и вот так нужно настроить
```q
 Настройка Shadow для DVIA-v2 (конкретно что включить)

Обязательно включи (ESSENTIAL HOOKS) - это весь первый блок


|**Enable Shadow**|✅ ВКЛЮЧИТЬ (главный тумблер)|
|**File System**|✅ ВКЛЮЧИТЬ|
|**Dynamic Libraries**|✅ ВКЛЮЧИТЬ|
|**URL Handlers**|✅ ВКЛЮЧИТЬ|
|**Environment Variables**|✅ ВКЛЮЧИТЬ|

✅ Рекомендуется включить (RECOMMENDED HOOKS)

|Опция|Действие|

|**Detection Frameworks**|✅ ВКЛЮЧИТЬ|
|**Foundation Framework**|✅ ВКЛЮЧИТЬ|

----------------------
----------------------
-----------------------
------------------------
--------------------
----------------------
------------------------

### ✅ Дополнительно (EXTRA HOOKS) — если не запустится

|Опция|Действие|

|**Mach Service Lookups**|✅ ВКЛЮЧИТЬ|
|**Runtime Symbol Lookups**|✅ ВКЛЮЧИТЬ|
|**Objective-C Class Methods**|✅ ВКЛЮЧИТЬ|
|**Anti-Debugging Methods**|✅ ВКЛЮЧИТЬ|


-------------------------
-------------------------
-------------------------
-------------------------
-------------------------


 ❌❌❌❌❌❌❌❌❌ НЕ ВКЛЮЧАЙ (❌DANGEROUS HOOKS❌)

Оставь **выключенными**:

- ❌Enforce App Sandbox
    
- ❌Low-Level File Handles
    
- ❌Hide Executable Memory
    
- ❌Hide Tweak Classes
    
- ❌Dynamic Library Loader
 Эти опции могут вызвать краш приложения на palera1n.
```

и это не помогло

------------

пробую через скрипт

`frida-ps -Uai` полное название приложения
     DVIA-v2       com.highaltitudehacks.DVIAswiftv2

сохраняю на стол скрипт

```js
cat > ~/Desktop/bypass.js << 'EOF'
if (ObjC.available) {
    var className = "JailbreakDetection";
    var methodName = "isJailbroken";
    var hook = ObjC.classes[className][methodName];
    Interceptor.attach(hook.implementation, {
        onLeave: function(retvalue) {
            console.log("[*] Original return value: " + retvalue);
            var newRetValue = ptr("0x0");
            retvalue.replace(newRetValue);
            console.log("[+] Return value replaced with: " + retvalue);
        }
    });
}
EOF
```

и вот так запускаю скрипт
```bash
frida -U -l ~/Desktop/bypass.js com.highaltitudehacks.DVIAswiftv2
```

но фрида не видит процесс, поэтому смысла нет,
во

========

короче - приложение падает из-за того, что не поддерживается моей ios 16


--------
