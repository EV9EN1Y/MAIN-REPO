# MASTG-TEST-0308: Runtime Use of Asymmetric Key Pairs Used For Multiple Purposes
https://mas.owasp.org/MASTG/tests/android/MASVS-CRYPTO/MASTG-TEST-0308/


Тест проверяет в рантайме, не используется ли одна и та же асимметричная ключевая пара для операций из разных групп. Например, если ключ, созданный для шифрования, используется еще и для подписи

Тест динамический и использует Frida для перехвата вызовов криптографических операций:

- `Cipher.init()` - для перехвата шифрования/дешифрования.
   
- `Signature.initSign()` и `initVerify()` - для перехвата подписи/верификации

| `Cipher.init()` (все оверлоады) |
| :------------------------------ |
| `Cipher.init(ENCRYPT_MODE)`     |
| `Cipher.init(DECRYPT_MODE)`     |
| `Signature.initSign()`          |
| `Signature.initVerify()`        |
|                                 |

   

Если один и тот же ключ будет использован и в `Cipher.init()` и в `Signature.initSign()`, тест провален


--------

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

скрипт с перехватом крипто функций

```q
cat > /Users/evgeniy/Desktop/hook_crypto_operations.js << 'EOF'
// hook_crypto_operations.js - Перехват криптографических операций
Java.perform(function() {
    console.log("[*] === Crypto Operations Hook Script Started ===");

    // 1. Перехват Cipher.init() — все режимы
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
            console.log("[+] Cipher.init() called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Key: " + key);
            console.log("[+] Key algorithm: " + key.getAlgorithm());
            console.log("[+] Key format: " + key.getFormat());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key);
        };
        console.log("[+] Hooked: Cipher.init(Key)");

        Cipher.init.overload('int', 'java.security.cert.Certificate').implementation = function(opmode, cert) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Certificate) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Certificate: " + cert);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, cert);
        };
        console.log("[+] Hooked: Cipher.init(Certificate)");

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
            console.log("[+] Key: " + key);
            console.log("[+] Params: " + params);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, key, params);
        };
        console.log("[+] Hooked: Cipher.init(Key, params)");

        Cipher.init.overload('int', 'java.security.cert.Certificate', 'java.security.spec.AlgorithmParameterSpec').implementation = function(opmode, cert, params) {
            var modeStr = "";
            switch(opmode) {
                case 1: modeStr = "ENCRYPT_MODE"; break;
                case 2: modeStr = "DECRYPT_MODE"; break;
                case 3: modeStr = "WRAP_MODE"; break;
                case 4: modeStr = "UNWRAP_MODE"; break;
                default: modeStr = "UNKNOWN (" + opmode + ")";
            }
            console.log("[+] Cipher.init(Certificate, params) called");
            console.log("[+] Mode: " + modeStr);
            console.log("[+] Certificate: " + cert);
            console.log("[+] Params: " + params);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.init(opmode, cert, params);
        };
        console.log("[+] Hooked: Cipher.init(Certificate, params)");
    } catch(e) {
        console.log("[-] Error hooking Cipher.init: " + e);
    }

    // 2. Перехват Signature.initSign() и initVerify()
    try {
        var Signature = Java.use("java.security.Signature");
        Signature.initSign.implementation = function(privateKey) {
            console.log("[+] Signature.initSign() called");
            console.log("[+] PrivateKey: " + privateKey);
            console.log("[+] Key algorithm: " + privateKey.getAlgorithm());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.initSign(privateKey);
        };
        console.log("[+] Hooked: Signature.initSign()");

        Signature.initVerify.overload('java.security.PublicKey').implementation = function(publicKey) {
            console.log("[+] Signature.initVerify(PublicKey) called");
            console.log("[+] PublicKey: " + publicKey);
            console.log("[+] Key algorithm: " + publicKey.getAlgorithm());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.initVerify(publicKey);
        };
        console.log("[+] Hooked: Signature.initVerify(PublicKey)");

        Signature.initVerify.overload('java.security.cert.Certificate').implementation = function(cert) {
            console.log("[+] Signature.initVerify(Certificate) called");
            console.log("[+] Certificate: " + cert);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.initVerify(cert);
        };
        console.log("[+] Hooked: Signature.initVerify(Certificate)");
    } catch(e) {
        console.log("[-] Error hooking Signature: " + e);
    }

    console.log("[*] === All crypto hooks are set. Waiting for operations... ===");
});
EOF
```

запуск скрипта

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_crypto_operations.js
```

Если один и тот же ключ будет использован и в `Cipher.init()`, и в `Signature.initSign()`, это будет означать провал теста



зупустил скрипт/приложение, анализирую результаты s



тест пройден ✅✅✅✅✅✅

### Статистика перехваченных вызовов

| Тип вызова | Количество |
|:-|-:|
| `Cipher.init()` (все оверлоады) | 642 |
| `Cipher.init(ENCRYPT_MODE)` | 202 |
| `Cipher.init(DECRYPT_MODE)` | 105 |
| `Signature.initSign()` | **0** |
| `Signature.initVerify()` | **0** |


Все 642 вызова `Cipher.init()` относятся к **симметричному шифрованию AES-GCM**:

- **EncryptedSharedPreferences** (AndroidX Security Crypto) - 307 вызовов (ENCRYPT + DECRYPT)
- **Google Tink** (`InsecureNonceAesGcmJce` / `AesGcmJce`) - ~320 вызовов для SecureStorage
- **SecureRandom** и прочие вспомогательные - ~15 вызовов

**Вызовов `Signature.initSign()` / `Signature.initVerify()` зафиксировано 0.** Асимметричная криптография в рантайме не используется. Все перехваченные ключи - симметричные (AES), что исключает возможность переиспользования одного ключа для разных групп операций

### ВЫВОД по тесту

Нарушений принципа разделения назначений ключей не обнаружено. Единственный используемый тип шифрования - AES-256-GCM. Асимметричные ключи и их cross-purpose использование отсутствуют


