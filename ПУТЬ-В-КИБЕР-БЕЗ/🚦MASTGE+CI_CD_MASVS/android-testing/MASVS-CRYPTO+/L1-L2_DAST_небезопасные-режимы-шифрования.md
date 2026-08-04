# MASTG-TEST-0350: Runtime Use of Broken Symmetric Encryption Modes
https://mas.owasp.org/MASTG/tests/android/MASVS-CRYPTO/MASTG-TEST-0350/

Тест проверяет в рантайме, не использует ли приложение небезопасные режимы шифрования (например, **ECB**) для защиты чувствительных данных. Даже если в коде есть вызов `Cipher.getInstance("AES/ECB/PKCS5Padding")`, но он никогда не вызывается для реальных данных, это не проблема. Тест же проверяет, вызывается ли он в реальных сценариях

Тест динамический и использует Frida для перехвата вызовов `Cipher.getInstance()` и `Cipher.init()` в реальном времени


Тест будет провален, если обнаружено использование режима ECB для шифрования данных в реальном времени

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

```



```q
cat > /Users/evgeniy/Desktop/hook_cipher_modes.js << 'EOF'
// hook_cipher_modes.js - Перехват Cipher.getInstance и Cipher.init
Java.perform(function() {
    console.log("[*] === Cipher Mode Hook Script Started ===");

    // 1. Перехват Cipher.getInstance() – все варианты
    try {
        var Cipher = Java.use("javax.crypto.Cipher");

        // Cipher.getInstance(String transformation)
        Cipher.getInstance.overload('java.lang.String').implementation = function(transformation) {
            console.log("[+] Cipher.getInstance(String) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            // Проверяем, не используется ли небезопасный режим ECB
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation);
        };
        console.log("[+] Hooked: Cipher.getInstance(String)");

        // Cipher.getInstance(String transformation, String provider)
        Cipher.getInstance.overload('java.lang.String', 'java.lang.String').implementation = function(transformation, provider) {
            console.log("[+] Cipher.getInstance(String, String) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Provider: " + provider);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation, provider);
        };
        console.log("[+] Hooked: Cipher.getInstance(String, String)");

        // Cipher.getInstance(String transformation, Provider provider)
        Cipher.getInstance.overload('java.lang.String', 'java.security.Provider').implementation = function(transformation, provider) {
            console.log("[+] Cipher.getInstance(String, Provider) called");
            console.log("[+] Transformation: " + transformation);
            console.log("[+] Provider: " + provider.getName());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            if (transformation && transformation.toUpperCase().indexOf("ECB") !== -1) {
                console.log("[!] WARNING: ECB mode detected in transformation: " + transformation);
            }
            return this.getInstance(transformation, provider);
        };
        console.log("[+] Hooked: Cipher.getInstance(String, Provider)");

    } catch(e) {
        console.log("[-] Error hooking Cipher.getInstance: " + e);
    }

    // 2. Перехват Cipher.init() чтобы видеть, когда шифр используется
    try {
        var Cipher = Java.use("javax.crypto.Cipher");
        Cipher.init.overload('int', 'java.security.Key').implementation = function(opmode, key) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Key) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Key algorithm: " + key.getAlgorithm());
            // Получаем текущий алгоритм и режим у шифра (если он уже инициализирован)
            var cipher = this;
            // Попытка получить параметры шифра (если доступны)
            try {
                var params = cipher.getParameters();
                console.log("[+] Cipher parameters: " + params);
            } catch(e) {}
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key);
        };
        console.log("[+] Hooked: Cipher.init(Key)");

        Cipher.init.overload('int', 'java.security.Key', 'java.security.spec.AlgorithmParameterSpec').implementation = function(opmode, key, params) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Key, params) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Key algorithm: " + key.getAlgorithm());
            console.log("[+] Params: " + params);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key, params);
        };
        console.log("[+] Hooked: Cipher.init(Key, params)");
    } catch(e) {
        console.log("[-] Error hooking Cipher.init: " + e);
    }

    console.log("[*] === All Cipher hooks are set. Waiting for crypto operations... ===");
});
EOF
```


запуск скрита

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_cipher_modes.js
```




результаты работы скрипта:


тест пройден  !!!

### Статистика перехваченных вызовов Cipher.getInstance()

| Алгоритм/Режим      | Кол-во | Источник                                                            |
| :------------------ | -----: | :------------------------------------------------------------------ |
| `AES/GCM/NoPadding` |     76 | Код приложения (`EncryptionUtil`)                                   |
| `AES/ECB/NoPadding` |     75 | Google Tink (`PrfAesCmac` - CMAC, часть EncryptedSharedPreferences) |
| `AES/CTR/NoPadding` |     25 | Google Tink (`AesSiv` - AES-SIV, часть EncryptedSharedPreferences)  |

### Анализ

**AES/GCM/NoPadding (76 вызовов)** - единственный режим, который использует само приложение через `EncryptionUtil`. Это безопасный AEAD-режим. 

**AES/ECB/NoPadding (75 вызовов)** - все вызовы идут из библиотеки **Google Tink**, а именно:
- `PrfAesCmac` – реализация AES-CMAC (NIST SP 800-38B), которая использует AES-ECB как строительный блок для вычисления MAC-кода
- Это часть `EncryptedSharedPreferences` (AndroidX Security Crypto), где AES-ECB применяется **не для шифрования данных**, а для криптографически корректного MAC-алгоритма
- Такой способ использования ECB - **стандартная и безопасная практика** (CMAC/S2V)

**AES/CTR/NoPadding (25 вызовов)** - все вызовы из Google Tink:
- `AesSiv` – реализация AES-SIV (RFC 5297), детерминированное аутентифицированное шифрование
- Используется `EncryptedSharedPreferences` для шифрования ключей SharedPreferences

### вывод по тесту

Приложение **не использует** небезопасные режимы шифрования для данных пользователя. Все вызовы ECB/CTR - внутренняя имплементация Google Tink, используемая корректно (CMAC + SIV). Непосредственно кодом приложения используется только `AES/GCM/NoPadding`


