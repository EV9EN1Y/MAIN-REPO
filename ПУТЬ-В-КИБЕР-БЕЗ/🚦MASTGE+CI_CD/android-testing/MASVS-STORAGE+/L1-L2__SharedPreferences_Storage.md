# MASTG-TEST-0287: Runtime Storage of Unencrypted Data via the SharedPreferences API
https://mas.owasp.org/MASTG/tests/android/MASVS-STORAGE/MASTG-TEST-0287/

Тест проверяет, не сохраняет ли приложение чувствительные данные (токены, пароли, ключи) через `SharedPreferences` в открытом виде. Даже если файл лежит в защищенной папке приложения, это не считается безопасным, так как при получении root-доступа или при создании бэкапа эти данные станут доступны злоумышленнику.

Тест динамический и использует Frida для перехвата вызовов методов записи в `SharedPreferences` в реальном времени


--------

буду фридой перехватывать вызовы SharedPreferences и смотреть че там внутри

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

```

скрипт для перехвата

```q
cat > /Users/evgeniy/Desktop/hook_shared_prefs.js << 'EOF'
// hook_shared_prefs.js - Перехват записи в SharedPreferences
Java.perform(function() {
    console.log("[*] === SharedPreferences Write Hook Script Started ===");

    // 1. Перехват SharedPreferences.Editor.putString()
    try {
        var Editor = Java.use("android.content.SharedPreferences$Editor");
        Editor.putString.implementation = function(key, value) {
            // Проверяем, не является ли значение слишком длинным для вывода
            var displayValue = value;
            if (value && value.length > 100) {
                displayValue = value.substring(0, 100) + "... (length: " + value.length + ")";
            }
            console.log("[+] putString() called");
            console.log("[+] Key: " + key);
            console.log("[+] Value: " + displayValue);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));

            // Вызываем оригинальный метод
            return this.putString(key, value);
        };
        console.log("[+] Hooked: SharedPreferences.Editor.putString()");
    } catch(e) {
        console.log("[-] Error hooking putString: " + e);
    }

    // 2. Перехват SharedPreferences.Editor.putStringSet()
    try {
        var Editor = Java.use("android.content.SharedPreferences$Editor");
        Editor.putStringSet.implementation = function(key, value) {
            var values = [];
            if (value) {
                var iterator = value.iterator();
                while (iterator.hasNext()) {
                    var val = iterator.next();
                    if (val && val.length > 50) {
                        values.push(val.substring(0, 50) + "...");
                    } else {
                        values.push(val);
                    }
                }
            }
            console.log("[+] putStringSet() called");
            console.log("[+] Key: " + key);
            console.log("[+] Values: " + JSON.stringify(values));
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));

            return this.putStringSet(key, value);
        };
        console.log("[+] Hooked: SharedPreferences.Editor.putStringSet()");
    } catch(e) {
        console.log("[-] Error hooking putStringSet: " + e);
    }

    // 3. Перехват SharedPreferences.Editor.apply() и commit()
    try {
        var Editor = Java.use("android.content.SharedPreferences$Editor");
        Editor.apply.implementation = function() {
            console.log("[+] apply() called - changes are being saved");
            return this.apply();
        };
        console.log("[+] Hooked: SharedPreferences.Editor.apply()");
    } catch(e) {
        console.log("[-] Error hooking apply: " + e);
    }

    try {
        var Editor = Java.use("android.content.SharedPreferences$Editor");
        Editor.commit.implementation = function() {
            console.log("[+] commit() called - changes are being saved synchronously");
            return this.commit();
        };
        console.log("[+] Hooked: SharedPreferences.Editor.commit()");
    } catch(e) {
        console.log("[-] Error hooking commit: " + e);
    }

    // 4. Дополнительно: перехват getString() чтобы видеть, что читается
    try {
        var SharedPreferences = Java.use("android.content.SharedPreferences");
        SharedPreferences.getString.implementation = function(key, defValue) {
            var value = this.getString(key, defValue);
            if (key && key.toLowerCase().indexOf("token") !== -1 || key && key.toLowerCase().indexOf("jwt") !== -1 || key && key.toLowerCase().indexOf("password") !== -1) {
                console.log("[+] getString() called for potentially sensitive key: " + key);
            }
            return value;
        };
        console.log("[+] Hooked: SharedPreferences.getString() (sensitive keys)");
    } catch(e) {
        console.log("[-] Error hooking getString: " + e);
    }

    console.log("[*] === All SharedPreferences hooks are set. Waiting for write operations... ===");
});
EOF

```

запускаю аппку со скрипта

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_shared_prefs.js
```


----

### вывод

в результатах работы скрипта не обнаружил каких-либо опасных, конфиденциальных данных!