# MASTG-TEST-0382: Runtime Use of Enforced Updating APIs
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0382/

**принудительно ли обновляется приложение**, и нельзя ли обойти это обновление

Приложения часто проверяют версию и требуют обновления, если она устарела. Это важно для безопасности, так как старые версии могут содержать уязвимости. Тест проверяет, работает ли этот механизм в реальном времени и можно ли его обойти

-------

то есть - для некоторых прил важно, чтобы приложение не продолжало норм работу и просило обновление!

----

нужно установить старую версию приложения - и перехватив запросы сетевые через burp или зап - смотреть - есть ли там проверки версий приложения, пересылает ли приложение свою версию на сервер - и сверяет ли оно это их!

и потом - проверить - получается ли подменить значения версии - и какая реакция на это

------

готовлю бупр

<img src="../../../assets/Снимок2026-07-1301.09.19.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





теперь нужно поставить на телефон ip мака в сети wifi общего (или так - ifconfig)

мой телефон onePlus8t android 14

настройки в телефоне - wifi - мой подключенный wifi - i (настройка)
настройка прокси - вручную - сервер - указать там ip wifi
порт 8080 - сохранить

в браузер перехожу http://192.168.0.15:8080

после чего - скачать на телефон и установить сертификат burp

он скачался в загрузки и от туда не устанавливается

нужно установить сертификат в системное хранилище (`System`). Это требует root-прав, и у меня есть Magisk

------

план такой - подготовить сертификат на ноуте, потом закинуть его в телефон - в системные файлы - и там его установить!

делее проверить, что трафик течет через burp моего ноута!

--------

**Подготовка сертификата на Mac**

```bash
# Перейти в папку с загрузками
cd ~/Downloads

# 1. Скачать сертификат Burp в DER-формате 
# wget http://192.168.0.105:8080/cert -O cacert.der

# 2. Конвертир DER в PEM
openssl x509 -inform DER -in cacert.der -out cacert.pem

# 3. Получить хеш для имени файла
CERT_HASH=$(openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1)
echo $CERT_HASH

# 4. Переименять файл в <хеш>.0
mv cacert.pem $CERT_HASH.0

```


кинуть этот файл на телефон

```sh
adb push $CERT_HASH.0 /sdcard/Download/
```



**Установить сертификат в систему через Magisk**

```shell
adb shell
su

# заменить 9a5ba575.0 на хеш, который я получил
CERT_NAME="9a5ba575.0"  # <-- ПОДСТАВИТЬ  ХЕШ
MODULE_NAME="burp_cert"

# папка для модуля Magisk
cd /data/adb/modules/
mkdir -p "$MODULE_NAME/system/etc/security/cacerts/"

# копир системные сертификаты в модуль
cp /system/etc/security/cacerts/* "$MODULE_NAME/system/etc/security/cacerts/"

# копир сертификат Burp в модуль
mv "/sdcard/Download/$CERT_NAME" "$MODULE_NAME/system/etc/security/cacerts/"

# правильные права
cd "$MODULE_NAME/system/etc/security/cacerts/"
chown root:root $CERT_NAME
chmod 644 $CERT_NAME

# Выход из shell
exit
exit

# Перезагружаем мобилку
adb reboot

снова ставлю прокси

WiFi → моя сеть → настройка → прокси вручную:
- Сервер: 192.168.0.11
- Порт: 8080
```

и наконец, проверить установку

После перезагрузки зайти в `Настройки` -> `Безопасность и конфиденциальность` -> `Другие настройки безопасности` -> `Доверенные учетные данные` -> вкладка `Система` (System)

должен буду увидеть там сертификат `PortSwigger CA`

все работает!

трафик с телефона пошел через мой бупр!





НО МОЕ ПРИЛОЖЕНИЕ [[0_MeetWay]]  ХОТЬ И ЗАПУСТИЛОСЬ, НО НЕ РАБОТАЕТ!
ВСЕ ПОТОМУ ЧТО ТАМ СТОИТ У МЕНЯ SSL Pinning!
который теперь нужно обойти!

но это цветочки, так как тут у меня стоит еще защита от отладки ))
которую тодде обойти нужно, благо, скрипты у меня есть!

--------


```js
// meetway-nuke-v2.js – ВЫЖИГАЕМ ВСЮ SSL ЗАЩИТУ
console.log("[*] MeetWay SSL Nuke v2");

Java.perform(function() {
    console.log("[*] Installing nuke hooks...");

    // ===== АНТИ-ДЕБАГ =====
    try { Java.use("android.os.Debug").isDebuggerConnected.implementation=function(){return false;}; } catch(e){}
    try { Java.use("java.lang.System").exit.implementation=function(c){}; } catch(e){}
    try { Java.use("android.os.Process").killProcess.implementation=function(p){}; } catch(e){}
    try { Java.use("com.evgeniy.meetway.util.SecurityDetector").isDeviceCompromised.implementation=function(c){return false;}; } catch(e){}
    console.log("[+] Anti-debug: OK");

    // ===== OkHttp CertificatePinner =====
    try {
        var CP = Java.use("okhttp3.CertificatePinner");
        for (var i = 0; i < CP.check.overloads.length; i++) {
            CP.check.overloads[i].implementation = function() {};
        }
        console.log("[+] CertificatePinner.check: ALL overloads nuked");
    } catch(e){ console.log("[-] CertPinner: " + e); }

    try {
        Java.use("okhttp3.OkHttpClient.Builder").certificatePinner.implementation = function(pinner) {
            return this; // игнорируем пиннер
        };
        console.log("[+] OkHttpClient.Builder.certificatePinner: blocked");
    } catch(e){}

    // ===== NetworkModule – ГЛАВНЫЙ ПИННЕР! =====
    try {
        var NM = Java.use("com.evgeniy.meetway.data.remote.NetworkModule");
        // Ленивая инициализация – перехватываем фабрику
        NM.createCertificatePinner.implementation = function() {
            console.log("[+] NetworkModule.createCertificatePinner: returning empty pinner");
            var Builder = Java.use("okhttp3.CertificatePinner$Builder");
            return Builder.$new().build();
        };
        // Геттер
        NM.getCertificatePinner.implementation = function() {
            console.log("[+] NetworkModule.getCertificatePinner: returning empty pinner");
            var Builder = Java.use("okhttp3.CertificatePinner$Builder");
            return Builder.$new().build();
        };
        console.log("[+] NetworkModule: REAL pinner nuked");
    } catch(e){ console.log("[-] NetworkModule: " + e); }

    // ===== SslPinningInterceptor =====
    try {
        var SPI = Java.use("com.evgeniy.meetway.util.SslPinningInterceptor");
        SPI.createCertificatePinner.implementation = function() {
            var B = Java.use("okhttp3.CertificatePinner$Builder");
            return B.$new().build();
        };
        SPI.createInterceptor.implementation = function() {
            var I = Java.use("okhttp3.Interceptor");
            return I.$new({intercept: function(c){return c.proceed(c.request());}});
        };
        console.log("[+] SslPinningInterceptor: OK");
    } catch(e){}

    // ===== MeetWayApp – getCertificatePinner =====
    try {
        Java.use("com.evgeniy.meetway.MeetWayApp").getCertificatePinner.implementation = function() {
            var B = Java.use("okhttp3.CertificatePinner$Builder");
            return B.$new().build();
        };
        console.log("[+] MeetWayApp.getCertificatePinner: OK");
    } catch(e){}

    // ===== HostnameVerifier =====
    try { Java.use("javax.net.ssl.HostnameVerifier").verify.implementation=function(h,s){return true;}; } catch(e){}
    console.log("[+] HostnameVerifier: OK");

    // ===== X509TrustManager =====
    try {
        var TM = Java.use("javax.net.ssl.X509TrustManager");
        TM.checkClientTrusted.overload('[Ljava.security.cert.X509Certificate;','java.lang.String').implementation=function(){};
        TM.checkServerTrusted.overload('[Ljava.security.cert.X509Certificate;','java.lang.String').implementation=function(){};
        console.log("[+] X509TrustManager: OK");
    } catch(e){}

    // ===== Conscrypt TrustManagerImpl =====
    try {
        var TMI = Java.use("com.android.org.conscrypt.TrustManagerImpl");
        for (var i = 0; i < TMI.checkServerTrusted.overloads.length; i++) {
            TMI.checkServerTrusted.overloads[i].implementation = function() { return; };
        }
        console.log("[+] TrustManagerImpl: " + TMI.checkServerTrusted.overloads.length + " overloads nuked");
    } catch(e){ console.log("[-] TMI: " + e); }

    console.log("[*] ===== ALL DONE =====");
});
```





Запуск:

```bash
  adb shell "am force-stop com.evgeniy.meetway"
  frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/meetway-nuke-v2.js
```

-------

короче, пока не получается полноценно перехватывать весь трафик,   проблема не в MeetWay, а в Android 14. 

Google начиная с 14-й версии запилил conscrypt
APEX - отдельный модуль для SSL, который:
- не использует /system/etc/security/cacerts/ (куда Magisk ложит сертификаты)
- использует свой изолированный стор
- read-only, mount не подходит
- TrustManagerImpl из этого модуля Frida хукает с трудом

Это проблема всех пентестеров на Android 14
!

Google так защищает ВСЕ приложения на Android 14!!!!!!!!!!!!!!!!!!!!!

conscrypt на Android 14 не пропускает Burp-сертификат на
уровне платформы. Но для r0capture вообще никаких преград нет - он BoringSSL на прямую в
памяти хукает


-------
### пробую r0capture

----

##  Результаты теста

Всё что планировал - сделал:

- Сертификат Burp установил через Magisk модуль ✅
- Прокси на телефоне настроил ✅
- Написал Frida скрипты для обхода анти-дебага и SSL пиннинга ✅
- Через Frida обошёл `NetworkModule`, `CertificatePinner`, `SslPinningInterceptor` ✅
- Добрался до нативного BoringSSL – перехватил `SSL_set_verify`, `SSL_CTX_set_custom_verify`, `SSL_get_verify_result` ✅

**НО** трафик через Burp так и не пошёл.

Проблема не в MeetWay, а в **Android 14**. Google вынес SSL/TLS в отдельный модуль (`com.android.conscrypt` APEX), который не использует `/system/etc/security/cacerts/`. У него свой изолированный список сертификатов, и Magisk туда не пробивается.

`curl` с телефона через Burp ходит нормально, а Java-приложения – нет. Падают с `Trust anchor for certification path not found`.

**Вывод:** на OnePlus 8T с Android 14 перехватить HTTPS трафик MeetWay через Burp не получилось. Не из-за защиты приложения (она там чисто символическая), а из-за платформы.

Для теста нужно либо устройство на Android 12-13, либо использовать r0capture

Скрипты для обхода сохранил на рабочем столе - `meetway-nuke-v2.js` и `meetway-nuclear.js`