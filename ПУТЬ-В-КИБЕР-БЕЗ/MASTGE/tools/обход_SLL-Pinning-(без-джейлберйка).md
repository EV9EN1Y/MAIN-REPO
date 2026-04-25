обход_SLL-Pinning без-джейлберйка с помощью фрида-гаджета впатченного в ipa

-----

буду выполнять тест на айфон 11 ios 26.1
без джейлбрейка
тест провожу на приложении 👉 [[0_MeetWay]] (соц сеть)

изначально там отсутствует SSL pinning

----------

### в данном тесте, я установлю в приложение SLL Pinning, затем через xCode установлю его и буду тестить

--------
# этап 1  (перехват без SSL pinnig)

здесь подробнее про перехват 👉 [[перехват-трафика-реал-айфона-через-WiFi_burp]]

и вот я спокойно могу перехватывать весь трафик своего приложения!

<img src="../../../assets/Снимок2026-04-2418.48.34.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



---------
# этап 2  (настраиваю SLL pinning в самом приложении MeetWay)

есть три базовых способа добавить sll pinning в приложение

#### 1)  способ от Apple - через Info.plist

просто в плист добавляю ключ
```swift
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSPinnedDomains</key>
    <dict>
        <key>api.mysite.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSPinnedCAIdentities</key>
            <array>
                <dict>
                    <key>SPKI-SHA256-BASE64</key>
                    <string>твой_хэш_публичного_ключа</string>
                </dict>
            </array>
        </dict>
    </dict>
</dict>
```

ключ здесь
```shell
openssl s_client -connect api.mysite.com:443 -servername api.mysite.com 2>&1 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
```

-------

#### 2)  способ ручной, добавить сертификат + сам код проверки

```swift
class SSLPinningDelegate: NSObject, URLSessionDelegate {
    func urlSession(_ session: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) {
        
        // Проверяем, что это server trust challenge
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Берём сертификат из бандла
        guard let certPath = Bundle.main.path(forResource: "my_certificate", ofType: "cer"),
              let localCertData = try? Data(contentsOf: URL(fileURLWithPath: certPath)),
              let localCert = SecCertificateCreateWithData(nil, localCertData as CFData) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Берём первый сертификат из цепочки сервера
        let serverCert = SecTrustCopyCertificateChain(serverTrust) as? [SecCertificate]
        guard let serverCertData = serverCert?.first.map(SecCertificateCopyData) as? Data else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        if serverCertData == localCertData as Data {
            // Всё ок, продолжаем
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            // Пиннинг не пройден — блокируем!
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

и везде - где использую сессии URL  - применяю проверку
```swift
let session = URLSession(configuration: .default, delegate: SSLPinningDelegate(), delegateQueue: nil)
```
#### 3)  способ 3 - через лайбру TrustKit

TrustKit - это open-source библиотека от Data Theorem. Она делает всё то же самое, что и предудущие способы, но с дополнительными плюшками: автоматический сваззлинг делегатов и reporting

это кайф , так как  -  если включить `kTSKSwizzleNetworkDelegates: true`, TrustKit сам подменит все `URLSession` делегаты. И даже не нужно менять существующий сетевой код, кра-со-та!

```swift
import TrustKit

// Настройка политики
let trustKitConfig = [
    kTSKSwizzleNetworkDelegates: true,
    kTSKPinnedDomains: [
        "api.mysite.com": [
            kTSKPublicKeyHashes: [
                "HXXQgxueCIU5TTLHob/bPbwcKOKw6DkfsTWYHbxbqTY=",
                "0SDf3cRToyZJaMsoS17oF72VMavLxj/N7WBNasNuiR8="
            ],
            kTSKIncludeSubdomains: true,
            kTSKEnforcePinning: true
        ]
    ]
] as [String : Any]

TrustKit.initSharedInstance(withConfiguration: trustKitConfig)
```

-------

### НАСТРАИВАЮ В ПРИЛОЖЕНИИ  TrustKit

поставлю лайбу через  Swift Package Manager 
https://github.com/datatheorem/TrustKit.git

далее в AppDelegate  подрубаю конфиг
```swift

let trustKitConfig = [
    kTSKSwizzleNetworkDelegates: true,   // магия — сам подменит все URLSession
    kTSKPinnedDomains: [
        "api.mysite.com": [              // твой домен
            kTSKPublicKeyHashes: [
                "HXXQgxueCIU5TTLHob/bPbwcKOKw6DkfsTWYHbxbqTY=",  // остновной ключ
                "WoiWRyIOVNa9ihaBciRSC7XHjliYS9VwUGOIud4PB18="   // резервный ключ
            ],
            kTSKEnforcePinning: true,    // жёстко блокируем, если хеш не совпал
            kTSKIncludeSubdomains: true   // защищаем все поддомены сразу
        ]
    ]
] as [String : Any]

TrustKit.initSharedInstance(withConfiguration: trustKitConfig)
```

далее, получаю ключи - **публичные ключи серверов** (SPKI хеши)

```shell
echo | openssl s_client -servername storage.yandexcloud.net -connect storage.yandexcloud.net:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=


echo | openssl s_client -servername api.yandex.ru -connect api.yandex.ru:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
2aEWzNRnJjQagTUqyiFmYeBS1AF+9pnrQvt0N3d6ER4=


echo | openssl s_client -servername firestore.googleapis.com -connect firestore.googleapis.com:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
+vLEyBQERqRwpgiGwEi7Dx6jujKTEdoJzr4CSmYXCz0=

```

подрубаю перехват burp, затем :
запускаю приложение:

и мое приложение больше не пропускает запросы от/на
`storage.yandexcloud.net` который я как раз - таки и указал в коде!

но вот запросы от functions.yandexcloud.net - пропускаются без проблем

но это проблема самого **TrustKit**  - он не может работать с YandexSDK, так как там используется **gRPC**

<img src="../../../assets/Снимок2026-04-2420.11.49.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

теперь я добавлю в код функции еще и этот домен:
```swift
"functions.yandexcloud.net": [
    kTSKPublicKeyHashes: [
        "Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=",
        "C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ="
    ],
    kTSKIncludeSubdomains: true,
    kTSKEnforcePinning: true
],
```
получаю хеш
```shell
echo | openssl s_client -servername functions.yandexcloud.net -connect functions.yandexcloud.net:443 2>/dev/null | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64
```
и все равно, не получается заблокировать запросы от/на functions.yandexcloud.net 

и вот почему:

```q
TrustKit не может перехватывать трафик, который не проходит через стандартные URLSession

### Почему так происходит:

1. Yandex SDK использует gRPC - это работает поверх HTTP/2, но имеет свой собственный TLS-handshake
    
2. gRPC на iOS часто использует низкоуровневые API - напрямую Network.framework или CFNetwork
    
3. TrustKit работает через swizzling делегатов URLSession - это не влияет на gRPC-соединения
```

хотя для `storage.yandexcloud.net` sll pinning работает успешно!!!


-------

# ИТОГОВАЯ НАСТРОЙКА SLL Pinning

короче: я решил сделать более универсальный 

добавил два класса для перехвата URLProtocol и для Swizzling методов

##### мой первый уровень защиты:

TrustKit в Appdelegate

- TrustKit делает **method swizzling** всех `URLSessionDelegate` методов
- и при каждом HTTPS запросе автоматически проверяет публичный ключ сервера
- сравнивает хеш ключа с заданными в конфиге
- при несовпадении блокирует соединение и вызывает делегат/нотификацию
##### мой второй уровень защиты класс GRPCInterceptor:
Ищет классы Yandex SDK
Делает swizzling их методов запросов 
В swizzled методе вызывает verifyPinning() 
Синхронно получает сертификат через URLSession
Сравнивает с теми же хешами
Возвращает true/false

##### мой второй уровень защиты класс GRPCURLProtocol
Проверяет хост и решает, перехватывать ли
принудительно извлекает сертификат
Сравнивает хеши
Принимает или отклоняет соединение

#### вот в этом файле - код SLL Pinning для моего приложения
# вот здесь можно посмотреть: 👉 [[Sll_Pinning_ios-meetWay]]

------

##  тестирую работу ssl pining:

до подключения sll pining - burp спокойно перехватывал все запросы , и я все видел в burp!

запустил приложение без перехвата через burp - и все прекрасно работает!!!
никаких проблем нет!

теперь отправляю весь трафик с телефона через ip ноутбука - далее через BURP suite - и приложение не запускается !!! 
<img src="../../../assets/Снимок2026-04-2423.14.14.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />


а если включить приложение без перехвата - оно запускается , и еще в процессе работы подключить перехват трафика через burp - то все, все вкладки пустые, не грузятся.... соединения рвутся ! ура! полностью работает защита от поддельных сертификатов!

<img src="../../../assets/Снимок2026-04-2423.09.18.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



старые смс не подгружаются

<img src="../../../assets/Снимок2026-04-2423.15.14.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



-------

## итого 
полностью исправно работает sll pinning!
✅ **Без Burp** - приложение работает нормально  
✅ **С Burp до запуска** - приложение не запускается  
✅ **С Burp во время работы** - данные не загружаются, вкладки пустые


##### кратко:
вот тут узнать про приложение: [[0_MeetWay]]
вот здесь про реализацию sll pinning в приложении [[Sll_Pinning_ios-meetWay]]
вот здесь - как перехватывать сеть [[перехват-трафика-реал-айфона-через-WiFi_burp]]


--------------------

## следующий шаг - ВЗЛОМАТЬ мою защиту sll pinning, перехватить и расшифровать трафик!

тест # MASTG-TEST-0091: Testing Reverse Engineering Tools Detection

-------

попробую теперь обойти свою защиту
защита - довольно сложная, местам кастомная

#### начальное положеине:

 айфон 11 ios 26 без джейлбрейка
 приложеине [[0_MeetWay]] установил через xCode

(то есть это упрощенный , легкий вариант, когда у меня уже есть ipa не шифрованный)

план такой:

1) ставлю приложение через xcode
2) скачиваю FridaGadget для айфона11
3) качаю ssl-kill-switch3
4) иду в бинарник приложения meetWay и патчу в него те две тулсы, что скачал выше
5) затем переподпишу приложеиние
6) ставлю на телефон пропатченное приложение, затем - пробую подключить перехват через Burp


------
установил приложение через xcode [[0_MeetWay]] (в приложении есть Sll pinnig) 

приложение установилось
иду в finder
cmd+shift+g
~/Library/Developer/Xcode/DerivedData

нахожу там основную бинарную папку MeetWay.app вес 180мб

смотрю содержимое папки
```q
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la
total 125504
-rwxr-xr-x   1 evgeniy  staff     35024 24 апр 23:05 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96 24 апр 19:31 _CodeSignature
drwxr-xr-x@ 16 evgeniy  staff       512 25 апр 15:10 .
drwx------@ 60 evgeniy  staff      1920 25 апр 15:12 ..
-rw-r--r--   1 evgeniy  staff     14412 25 апр 15:10 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132 25 апр 15:10 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696 25 апр 15:10 Assets.car
-rw-r--r--   1 evgeniy  staff     15136 24 апр 19:30 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864 24 апр 19:31 Frameworks
-rw-r--r--   1 evgeniy  staff       996 24 апр 19:30 GoogleService-Info.plist
-rw-r--r--   1 evgeniy  staff      3717 24 апр 19:30 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664 25 апр 15:10 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41387504 25 апр 15:10 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8 24 апр 19:30 PkgInfo
drwxr-xr-x   4 evgeniy  staff       128 24 апр 19:33 TrustKit_TrustKit.bundle
-rw-r--r--   1 evgeniy  staff     49852 24 апр 19:30 words.txt
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```

из этих папок меня интересует главный бинарник 
91664 25 апр 15:10 MeetWay

в него нужно будет заинжектить  FridaGadget 

##### план  
```shell
# 1. Скачай insert_dylib (если ещё нет)
git clone https://github.com/tyilo/insert_dylib
cd insert_dylib
xcodebuild

cp build/Release/insert_dylib /usr/local/bin/

# 2. скачать и разметить FridaGadget
//качаю
curl -L -O https://github.com/frida/frida/releases/download/17.9.1/frida-gadget-17.9.1-ios-universal.dylib.gz

// расспакую/именую
gunzip -k frida-gadget-17.9.1-ios-universal.dylib.gz && mv frida-gadget-17.9.1-ios-universal.dylib frida-gadget.dylib

// проверю, что все ок
ls -lh frida-gadget.dylib


//⭐️🟢😈инжектирую FridaGadget в бинарник MeetWay
insert_dylib --strip-codesig --inplace \
  @executable_path/Frameworks/frida-gadget.dylib \
  /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/MeetWay
  
  ---------
  копирую в сам бинарник
  
  mkdir -p /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/Frameworks && \
cp ~/insert_dylib/frida-gadget.dylib /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/Frameworks/
  
  --------
ТЕПЕРЬ НУЖНО ПЕРЕПОДПИСАТЬ

переподпись фридаГаджета

codesign -f -s "Apple Development" /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/Frameworks/frida-gadget.dylib

переподпись приложения

codesign -f -s "Apple Development" /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app
  
  проверочка что все на месте
  codesign -dv /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app
  
  вижу Format=app bundle with Mach-O thin (arm64) - все окей!!
  
  
```


ТЕПЕРЬ НУЖНО ЗАПУСТИТЬ ДАННОЕ ПРИЛОЖЕНИЕ ЧЕРЕЗ xcode

то есть - я сперва запустил приложение через xcode - перепатчил его и переподписал (вес бинарной папки увеличился на 30 мб ), затем я запускаю тут же это приложение

теперь после запуска приложения
проверяю через фриду - процесс
`frida -U MeetWay`


но не вижу там гаджета (((
```
evgeniy@Evgeniys-MacBook-Pro insert_dylib % frida -U MeetWay
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
   . . . .   Connected to iPhone (id=00008030-000149683EFB802E)
Failed to attach: unable to attach to the specified process
evgeniy@Evgeniys-MacBook-Pro insert_dylib %
```

проверю что фрида вообще видит процессы 
`frida-ps -Uai`

и да, приложение видит она
1485   MeetWay                  AIVARO22-2025-1.0


нужно обязательно сказать фриде на маке - то - где находится , в то место, в котором фрида его ждет по умолчанию
```shell
mkdir -p ~/.cache/frida && cp ~/insert_dylib/frida-gadget.dylib ~/.cache/frida/gadget-ios.dylib
```

теперь подключаюсь к приложению
```shell
frida -U MeetWay
```

и вуаля ! вижу процесс
```q
evgeniy@Evgeniys-MacBook-Pro ~ % frida -U MeetWay
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
   . . . .   Connected to iPhone (id=00008030-000149683EFB802E)

[iPhone::MeetWay ]->
```

##### ОТЛИЧНО! фрида гаджет успешно установлен и коннектится с фридой на маке!

теперь нужно загрузить `ssl-kill-switch3` но, нигде не могу найти готовое
поэтому файл сам создам
```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
// SSL Kill Switch
var sec = Process.findModuleByName("Security");
if (sec) {
    var exports = sec.enumerateExports();
    exports.forEach(function(exp) {
        if (exp.name === "_SecTrustEvaluate") {
            Interceptor.attach(exp.address, {
                onLeave: function(retval) {
                    retval.replace(0);
                }
            });
            console.log("[*] Patched _SecTrustEvaluate");
        }
        if (exp.name === "_SecTrustEvaluateWithError") {
            Interceptor.attach(exp.address, {
                onLeave: function(retval) {
                    retval.replace(1);
                }
            });
            console.log("[*] Patched _SecTrustEvaluateWithError");
        }
    });
    console.log("[SSL Kill Switch] Loaded!");
} else {
    console.log("[!] Security framework not found");
}
ENDOFSCRIPT
```



теперь снова подключаюсь 
`      frida -U MeetWay      `

и после подключения к фрид-гаджету - 
подрубаю скриптик
`    %load ssl-kill-switch3.js     `

все успешно подгрузилось!
[SSL Kill Switch] Loaded!
[iPhone::MeetWay ]->


<img src="../../../assets/Снимок2026-04-2516.28.09.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




быстро подрубаю перехват через Burp
в Burp порт 8080 и any device 
в айфоне - в wifi ставлю IP который на маке

и вуаля , трафик телефона идет через burp
(зашел в сафари, вижу, что трафик перехватывается впринципе)

захожу в приложение MeetWay
и...
ТРАФИК НЕ ИДЕТ НИКУДА, и ВСЕ ЗАПРОСЫ СЕТЕВЫЙ ВНУТРИ ПРИЛОЖЕНИЯ ПАДАЮТ, ЗНАЧИТ - sll pinning задетектил неверные сертификаты и палит контору!
обойти sll pinning не получилось...

разбираемся в текущем положении:
смотрю текущие модули:
```shell
[iPhone::MeetWay ]-> Process.enumerateModules().filter(m => m.name.toLowerCase().includes("
grpc") || m.name.toLowerCase().includes("trustkit") || m.name.toLowerCase().includes("yande
x")).map(m => m.name)
[
    "YandexLoginSDK",
    "grpc",
    "grpcpp",
    "openssl_grpc",
    "TrustKit"
]
[iPhone::MeetWay ]->
```

мой предыдущий скрипт прямо не работает, так как - он расчитан на системные методы, а здесь используются кастомные TrustKit и gRPC - которые используют свои собственные механизмы проверки сертификатов, минуя системные SecTrustEvaluate

поэтому делаю новый скрипт, который будет отключать конкретно эти модули:

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
// SSL Kill Switch — TrustKit + gRPC

console.log("[SSL Kill Switch] Starting...");

// 1. Вырубаем TrustKit
if (ObjC.available) {
    var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
    if (TSKPinningValidator) {
        console.log("[*] Found TSKPinningValidator");

        var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
        if (evaluateTrust) {
            Interceptor.attach(evaluateTrust.implementation, {
                onLeave: function(retval) {
                    retval.replace(ptr(1));
                    console.log("[*] TrustKit bypassed");
                }
            });
        }
    }
}

// 2. Вырубаем gRPC через openssl_grpc
var openssl_grpc = Process.findModuleByName("openssl_grpc");
if (openssl_grpc) {
    console.log("[*] Found openssl_grpc");
    
    var SSL_set_verify = Module.findExportByName("openssl_grpc", "SSL_set_verify");
    if (SSL_set_verify) {
        Interceptor.attach(SSL_set_verify, {
            onEnter: function(args) {
                args[0] = ptr(0);
            }
        });
        console.log("[*] gRPC SSL_set_verify patched");
    }
}

console.log("[SSL Kill Switch] All hooks installed!");
ENDOFSCRIPT
```
переподключаюсь
`frida -U MeetWay`

подгружаю скрипт
`%load ssl-kill-switch3.js`

получаю ошибку в строке 28 скрипта 

гляну процссы
```shell
Process.findModuleByName("openssl_grpc").enumerateExports().filter(e => e.name.toLowerCase().includes("ssl")).map(e => e.name)
```

и вижу, просто ебическую  гору процессов..... 1300 строк процессов

```c
[
    "GRPC_BIO_f_ssl",
    "GRPC_BIO_set_ssl",
    "GRPC_BORINGSSL_keccak",
    "GRPC_BORINGSSL_keccak_absorb",
    "GRPC_BORINGSSL_keccak_init",
    "GRPC_BORINGSSL_keccak_squeeze",
    "GRPC_BORINGSSL_self_test",
    "GRPC_ERR_load_SSL_strings",
    "GRPC_OPENSSL_add_all_algorithms_conf",
    "GRPC_OPENSSL_asprintf",
    "GRPC_OPENSSL_calloc",
    "GRPC_PEM_write_bio_SSL_SESSION",
    "GRPC_RAND_OpenSSL",
    "GRPC_SSL_CIPHER_is_block_cipher",
    "GRPC_SSL_CIPHER_standard_name",
    "GRPC_SSL_COMP_add_compression_method",
    "GRPC_SSL_CREDENTIAL_free",
    "GRPC_SSL_CTX_use_psk_identity_hint",
    "GRPC_SSL_ECH_KEYS_add",
    "GRPC_SSL_ECH_KEYS_up_ref",
    "GRPC_SSL_SESSION_copy_without_early_data",
    "GRPC_SSL_accept",
    "GRPC_SSL_add0_chain_cert",
    "GRPC_SSL_get1_session",
    "GRPC_SSL_get_SSL_CTX",
    "GRPC_SSL_get_all_cipher_names",
    "GRPC_SSL_has_pending",
    "GRPC_SSL_set_read_ahead",
    "GRPC_SSL_set_renegotiate_mode",
    
    "OPENSSL_get_armcap",
    "OPENSSL_get_armcap_pointer_for_test",
    "_ZN10ssl_ctx_stC1EPK13ssl_method_st",
    "_ZN14ssl_session_stD2Ev",
    "_ZN17ssl_credential_stD1Ev","_ZN4bssl4GRPC10MakeUniqueINS0_9TicketKeyEJEEENSt3__110unique_ptrIT_NS0_8internal7DeleterEEEDpOT0_",
    "_ZN4bssl4GRPC10SSL3_STATEC1Ev",
    "GRPC_SSL_set_verify",
    "GRPC_SSL_set_verify_algorithm_prefs",
    "GRPC_SSL_set_verify_depth",
    "GRPC_SSL_set_wfd",
    "_ZN4bssl4GRPC11DTLS1_STATED1Ev"
    "_ZN4bssl4GRPC18ssl_set_read_errorEP6ssl_st",
    "_ZN6ssl_stC2EP10ssl_ctx_st",
    "_ZN6ssl_stD1Ev",
    
    и так 1300 строк
    
    "openssl_grpcVersionNumber",
    "openssl_grpcVersionString"
]
    
```

нужна мне вот эта - `GRPC_SSL_CTX_set_custom_verify` - именно она отвечает за проверку сертификатов в gRPC

гляну - какие функции доступны
```q
Process.findModuleByName("openssl_grpc").enumerateExports().filter(e => e.name === "GRPC_SSL_CTX_set_custom_verify" || e.name === "GRPC_SSL_CTX_set_verify" || e.name === "GRPC_SSL_set_verify").map(e => e.name + " -> " + e.address)

0--------------
вижу

[
    "GRPC_SSL_CTX_set_custom_verify -> 0x10ce7de08",
    "GRPC_SSL_CTX_set_verify -> 0x10ce8fdbc",
    "GRPC_SSL_set_verify -> 0x10ce8fc90"
]
[iPhone::MeetWay ]->
```

меняю скрипт

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
// SSL Kill Switch — TrustKit + gRPC v3

console.log("[SSL Kill Switch] Starting...");

// 1. TrustKit
if (ObjC.available) {
    var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
    if (TSKPinningValidator) {
        var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
        if (evaluateTrust) {
            Interceptor.attach(evaluateTrust.implementation, {
                onLeave: function(retval) {
                    retval.replace(ptr(1));
                }
            });
            console.log("[*] TrustKit hooked");
        }
    }
}

// 2. gRPC — через enumerateExports
var mod = Process.findModuleByName("openssl_grpc");
if (mod) {
    mod.enumerateExports().forEach(function(exp) {
        if (exp.name === "GRPC_SSL_CTX_set_custom_verify" ||
            exp.name === "GRPC_SSL_CTX_set_verify" ||
            exp.name === "GRPC_SSL_set_verify") {
            Interceptor.attach(exp.address, {
                onEnter: function(args) {
                    console.log("[*] gRPC verify hooked: " + exp.name);
                    args[1] = ptr(0);
                }
            });
            console.log("[*] Hooked " + exp.name);
        }
    });
}

console.log("[SSL Kill Switch] All hooks installed!");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

и вроде бы все заебись...
```
evgeniy@Evgeniys-MacBook-Pro ~ % frida -U MeetWay
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
   . . . .   Connected to iPhone (id=00008030-000149683EFB802E)

[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] Starting...
[*] TrustKit hooked
[*] Hooked GRPC_SSL_CTX_set_custom_verify
[*] Hooked GRPC_SSL_CTX_set_verify
[*] Hooked GRPC_SSL_set_verify
[SSL Kill Switch] All hooks installed!
[iPhone::MeetWay ]->
```

но приложеине зависает и не реагирует на касания вообще...

то есть - когда я отключаю функции проверок сертификатов - отрубается и сам функционал приложения, то есть зависает...

нужно понять, как и почему все падает, как работают функции, может можно подменять значения которые функции возвращают...

пробую убрать часть хуков
```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
// SSL Kill Switch — только TrustKit

console.log("[SSL Kill Switch] Starting...");

if (ObjC.available) {
    var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
    if (TSKPinningValidator) {
        var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
        if (evaluateTrust) {
            Interceptor.attach(evaluateTrust.implementation, {
                onLeave: function(retval) {
                    retval.replace(ptr(1));
                }
            });
            console.log("[*] TrustKit hooked");
        }
    }
}

// gRPC хуки комментим — приложение с ними зависает
console.log("[SSL Kill Switch] Loaded!");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

но приложение тоже виснет и от такого скрипта

---------

выясняю

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] Test — no hooks");
ENDOFSCRIPT

от простого - такого скрипта - приложеине не виснет
```

проверяю **TrustKit** , падает ли тольк от него приложение
```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit test...");
if (ObjC.available) {
    var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
    if (TSKPinningValidator) {
        var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
        if (evaluateTrust) {
            Interceptor.attach(evaluateTrust.implementation, {
                onLeave: function(retval) {
                    console.log("[*] TrustKit called, retval = " + retval);
                }
            });
            console.log("[*] TrustKit logging hook set");
        }
    }
}
console.log("[SSL Kill Switch] Logging only");
ENDOFSCRIPT
```
и да - приложение зависает от этого скрипта!!!

-------

попробую **обернуть в try-catch** и использовать **слабую ссылку** на метод

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit safe hook...");

if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            console.log("[*] Found TSKPinningValidator");
            
            // Пробуем другой метод
            var methods = TSKPinningValidator.$ownMethods;
            console.log("[*] Available methods:");
            methods.forEach(function(m) {
                console.log("    " + m);
            });
            
            // Хукаем evaluateTrust:forHostname: максимально аккуратно
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onEnter: function(args) {
                        console.log("[*] evaluateTrust called");
                    },
                    onLeave: function(retval) {
                        console.log("[*] evaluateTrust returned: " + retval);
                        // Пока не трогаем retval
                    }
                });
                console.log("[*] evaluateTrust hook installed");
            }
        }
    } catch(e) {
        console.log("[!] Error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Done");
ENDOFSCRIPT
```
переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

```
приложение - все равно зависло

[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] TrustKit safe hook...
[*] Found TSKPinningValidator
[*] Available methods:
    + allowsAdditionalTrustAnchors
    - initWithDomainPinningPolicies:hashCache:ignorePinsForUserTrustAnchors:validationCallbackQueue:validationCallback:
    - evaluateTrust:forHostname:
    - handleChallenge:completionHandler:
    - spkiHashCache
    - setSpkiHashCache:
    - domainPinningPolicies
    - ignorePinsForUserTrustAnchors
    - validationCallback
    - validationCallbackQueue
    - .cxx_destruct
[*] evaluateTrust hook installed
[SSL Kill Switch] Done
[iPhone::MeetWay ]->
```

я понял, в чем проблема, я сейчас отключил фриду в терминале, но приложение не вырубал (оно было зависшее), но после отключении фриды - приложеине заработало - и автоматически поочереди выполнились все процессы, что там были сетевые (они как в очереди стояли, и ждали видимо ответ от функций sll pinning (пускать трафик - или нет), видимо , нужно понять, что именно, должны возвращать эти функции для успешного пропуска трафика)

----------

для TrustKit попробую сделать - так, чтобы возвращал хук значение - 1

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit — force trust...");

if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        console.log("[*] evaluateTrust — forcing YES");
                        // Заменяем возвращаемое значение на YES (1)
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit evaluateTrust hooked");
            }
            
            // Также хукнем handleChallenge — он тоже может блокировать
            var handleChallenge = TSKPinningValidator['- handleChallenge:completionHandler:'];
            if (handleChallenge) {
                Interceptor.attach(handleChallenge.implementation, {
                    onEnter: function(args) {
                        console.log("[*] handleChallenge — forcing accept");
                        // Вызываем completionHandler с accept
                        var completion = new ObjC.Object(args[3]);
                        completion(0, 2); // NSURLSessionAuthChallengeUseCredential
                    }
                });
                console.log("[*] TrustKit handleChallenge hooked");
            }
        }
    } catch(e) {
        console.log("[!] Error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Done");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

о!! приложеиние - не падает местами!

отлично!! вижу -  evaluateTrust — forcing YES
```js
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] TrustKit — force trust...
[*] TrustKit evaluateTrust hooked
[*] TrustKit handleChallenge hooked
[SSL Kill Switch] Done
[iPhone::MeetWay ]-> [*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
```

-----

нужно добавить **gRPC-трафик**

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit evaluateTrust only...");

if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        console.log("[*] evaluateTrust — forcing YES");
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit hooked");
            }
        }
    } catch(e) {
        console.log("[!] Error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Done");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

кайф - теперь оно не падает, но и не пропускает весь трафик
```
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] TrustKit evaluateTrust only...
[*] TrustKit hooked
[SSL Kill Switch] Done
[iPhone::MeetWay ]-> [*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
[*] evaluateTrust — forcing YES
```
отлично, TrustKit-часть теперь работает идеально! Приложение не виснет, хуки срабатывают. Но трафика нет, потому что **gRPC-соединения до сих пор проверяют сертификаты** и блокируют трафик

Теперь аккуратно добавим gRPC-хуки, используя тот же принцип - `onLeave` с возвратом правильного значения

--------

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit + gRPC...");

// 1. TrustKit
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit hooked");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

// 2. gRPC — через GRPC_SSL_set_verify (НЕ трогаем CTX_set)
var mod = Process.findModuleByName("openssl_grpc");
if (mod) {
    mod.enumerateExports().forEach(function(exp) {
        if (exp.name === "GRPC_SSL_set_verify") {
            Interceptor.attach(exp.address, {
                onEnter: function(args) {
                    // Сохраняем оригинальный args[1] (verify_mode)
                    this.origMode = args[1];
                    // Устанавливаем SSL_VERIFY_NONE (0)
                    args[1] = ptr(0);
                }
            });
            console.log("[*] GRPC_SSL_set_verify hooked");
        }
    });
    console.log("[*] gRPC module found");
}

console.log("[SSL Kill Switch] Done");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

приложеине - не зависает - работает!! но и трафик не идет по нему не с перехватом burp , не без него

--------

```js
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit + NSURLProtocol...");

// 1. TrustKit
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit hooked");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

// 2. NSURLSession — разрешаем прокси
if (ObjC.available) {
    try {
        var NSURLSessionConfiguration = ObjC.classes.NSURLSessionConfiguration;
        if (NSURLSessionConfiguration) {
            var defaultConfig = NSURLSessionConfiguration.defaultSessionConfiguration();
            if (defaultConfig) {
                defaultConfig.setConnectionProxyDictionary_({
                    "HTTPEnable": 1,
                    "HTTPProxy": "192.168.0.100", //  IP  мака
                    "HTTPPort": 8080,
                    "HTTPSEnable": 1,
                    "HTTPSProxy": "192.168.0.100", //  IP  мака
                    "HTTPSPort": 8080
                });
                console.log("[*] Proxy configured for NSURLSession");
            }
        }
    } catch(e) {
        console.log("[!] Proxy error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Done");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

и тоже самое, виснет, трафик даже внутри приложения безе перехвата не идет


------------


видимо, нужно узнать больше данных!
понаблюдать за процессами динамически, понять - что возвращают они!
соберу логи!

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Watch] Starting observation...");

// 1. Логируем TrustKit без модификаций
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onEnter: function(args) {
                        var trust = new ObjC.Object(args[2]);
                        var hostname = new ObjC.Object(args[3]);
                        console.log("[TrustKit] evaluateTrust called");
                        console.log("  hostname: " + hostname.toString());
                    },
                    onLeave: function(retval) {
                        console.log("  returns: " + retval);
                    }
                });
                console.log("[*] TrustKit watch active");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

// 2. Логируем gRPC вызовы
var mod = Process.findModuleByName("openssl_grpc");
if (mod) {
    mod.enumerateExports().forEach(function(exp) {
        if (exp.name === "GRPC_SSL_CTX_set_custom_verify" ||
            exp.name === "GRPC_SSL_CTX_set_verify" ||
            exp.name === "GRPC_SSL_set_verify") {
            Interceptor.attach(exp.address, {
                onEnter: function(args) {
                    console.log("[gRPC] " + exp.name + " called");
                    console.log("  mode: " + args[1]);
                    if (args[2]) console.log("  callback: " + args[2]);
                }
            });
            console.log("[*] gRPC watching: " + exp.name);
        }
    });
}

console.log("[SSL Watch] All watchers active — observe and report!");
ENDOFSCRIPT
```



переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

И ПРИЛОЖЕНИЕ ПОЛНОСТЬЮ ВИСНЕТ
ДАЖЕ **простое наблюдение за gRPC-функциями вешает приложение**! Значит, любое вмешательство в `openssl_grpc` критично - даже `Interceptor.attach` без модификации ломает внутреннюю логику gRPC


----

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Watch] Safe observation...");

// 1. TrustKit — только логирование
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onEnter: function(args) {
                        var hostname = new ObjC.Object(args[3]);
                        console.log("[TrustKit] hostname: " + hostname.toString());
                    },
                    onLeave: function(retval) {
                        console.log("  returns: " + retval);
                    }
                });
                console.log("[*] TrustKit watch active");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

// 2. NSURLSession delegate — без gRPC
if (ObjC.available) {
    try {
        var NSURLSession = ObjC.classes.NSURLSession;
        if (NSURLSession) {
            console.log("[*] NSURLSession found");
        }
    } catch(e) {}
}

console.log("[SSL Watch] Ready — no gRPC interference");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

приложение не виснет, все окей
перехватил методы
```
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Watch] Safe observation...
[*] TrustKit watch active
[*] NSURLSession found
[SSL Watch] Ready — no gRPC interference
[iPhone::MeetWay ]-> [TrustKit] hostname: functions.yandexcloud.net
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: baket-ivaaro.storage.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0
[TrustKit] hostname: functions.yandexcloud.net
  returns: 0x0

```
че известно:

1. **ВСЕ запросы** идут через TrustKit - даже `functions.yandexcloud.net` ( который   думал, что идёт через gRPC)!
   
2. **Все возвращают `0x0`** - это значит `NO`/`false`, то есть проверка НЕ пройдена. Но приложение работает без Burp, значит `0x0` - это нормальный ответ "сертификат не проверен через пиннинг", а не "заблокировать"
   
3. Домены: `functions.yandexcloud.net` и `baket-ivaaro.storage.yandexcloud.net`
   

Но самое важное: **gRPC-трафик тоже проходит через TrustKit**! А мы раньше думали, что TrustKit его не видит

Когда мы **форсим `0x1` (YES)** через `retval.replace(ptr(1))`, приложение работает, но трафик не идёт через прокси, потому что в случае с Burp TrustKit **не вызывается вообще** - NSURLSession просто не может установить соединение.

Нам нужно не TrustKit обходить, а **NSURLSession** заставить принимать любые сертификаты

скрипт вообще не трогает TrustKit и openssl_grpc, а перехватывает вызов делегата NSURLSession и **принимает любой сертификат**
```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] NSURLSession delegate approach...");

if (ObjC.available) {
    // Перехватываем URLSession:didReceiveChallenge:completionHandler:
    var NSObject = ObjC.classes.NSObject;
    if (NSObject) {
        var didReceiveChallenge = NSObject['- URLSession:didReceiveChallenge:completionHandler:'];
        if (didReceiveChallenge) {
            Interceptor.attach(didReceiveChallenge.implementation, {
                onEnter: function(args) {
                    // args[0] = self
                    // args[1] = selector
                    // args[2] = NSURLSession
                    // args[3] = NSURLAuthenticationChallenge
                    // args[4] = completionHandler block
                    try {
                        var challenge = new ObjC.Object(args[3]);
                        var protectionSpace = challenge.protectionSpace();
                        var authMethod = protectionSpace.authenticationMethod();
                        
                        if (authMethod.toString() === "NSURLAuthenticationMethodServerTrust") {
                            console.log("[*] Server trust challenge — accepting");
                            var trust = protectionSpace.serverTrust();
                            var credential = ObjC.classes.NSURLCredential.credentialForTrust_(trust);
                            var completionHandler = new ObjC.Object(args[4]);
                            completionHandler(0, credential); // NSURLSessionAuthChallengeUseCredential
                        }
                    } catch(e) {
                        console.log("[!] Error: " + e.message);
                    }
                }
            });
            console.log("[*] URLSession delegate hooked");
        }
    }
}

console.log("[SSL Kill Switch] Ready");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `


результат: (первая маленькая победа)
приложение не виснет, и при отключенном burp - трафик спокойно идет и приложение работает как нужно, НО ПРИ ПОДКЛЮЧЕНИИ burp - срабатывает sll pinning и трафик обраывается.  

```
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] NSURLSession delegate approach...
[SSL Kill Switch] Ready
[iPhone::MeetWay ]->
```

TrustKit **вызывается для ВСЕХ доменов** и возвращает `0x0`. При этом без Burp всё работает. Значит `0x0` - это видимо    "сертификат не подходит под пиннинг, но системное доверие ещё не проверено"

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit force YES + proxy configuration...");

// 1. Настраиваем прокси для NSURLSession
if (ObjC.available) {
    try {
        var config = ObjC.classes.NSURLSessionConfiguration.ephemeralSessionConfiguration();
        config.setConnectionProxyDictionary_({
            "HTTPEnable": 1,
            "HTTPProxy": "192.168.0.100",
            "HTTPPort": 8080,
            "HTTPSEnable": 1,
            "HTTPSProxy": "192.168.0.100",
            "HTTPSPort": 8080
        });
        // Делаем её конфигом по умолчанию
        ObjC.classes.NSURLSessionConfiguration.$ownMethods
            .filter(function(m) { return m.includes('defaultSessionConfiguration'); })
            .forEach(function(m) {
                try {
                    Interceptor.attach(ObjC.classes.NSURLSessionConfiguration[m].implementation, {
                        onLeave: function(retval) {
                            var cfg = new ObjC.Object(retval);
                            cfg.setConnectionProxyDictionary_({
                                "HTTPEnable": 1,
                                "HTTPProxy": "192.168.0.100",
                                "HTTPPort": 8080,
                                "HTTPSEnable": 1,
                                "HTTPSProxy": "192.168.0.100",
                                "HTTPSPort": 8080
                            });
                        }
                    });
                } catch(e) {}
            });
        console.log("[*] Proxy hooked");
    } catch(e) {
        console.log("[!] Proxy error: " + e.message);
    }
}

// 2. TrustKit форсим YES
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit forced YES");
            }
        }
    } catch(e) {}
}

console.log("[SSL Kill Switch] Ready");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

результат, приложение не виснет, но и трафик не пропускается вообще, и без перехвата тоже трафик не работает, и с перехватом не работает


------


пробую добавить логирование
- Какие методы есть у NSURLSessionConfiguration
   
- Какие домены проверяет TrustKit
   
- Че возвращает TrustKit до нашей модификации

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit force YES + proxy with LOGS...");

// 1. Настраиваем прокси для NSURLSession
if (ObjC.available) {
    try {
        console.log("[*] Setting up proxy...");
        var config = ObjC.classes.NSURLSessionConfiguration.ephemeralSessionConfiguration();
        console.log("[*] Config class: " + config.$className);
        
        config.setConnectionProxyDictionary_({
            "HTTPEnable": 1,
            "HTTPProxy": "192.168.0.100",
            "HTTPPort": 8080,
            "HTTPSEnable": 1,
            "HTTPSProxy": "192.168.0.100",
            "HTTPSPort": 8080
        });
        console.log("[*] Proxy dict set on ephemeral config");
        
        // Логируем ВСЕ методы defaultSessionConfiguration
        var methods = ObjC.classes.NSURLSessionConfiguration.$ownMethods;
        console.log("[*] NSURLSessionConfiguration methods:");
        methods.forEach(function(m) {
            console.log("    " + m);
        });
        
        console.log("[*] Proxy setup done");
    } catch(e) {
        console.log("[!] Proxy error: " + e.message);
        console.log("[!] Stack: " + e.stack);
    }
}

// 2. TrustKit форсим YES с логами
if (ObjC.available) {
    try {
        console.log("[*] Looking for TrustKit...");
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            console.log("[*] TSKPinningValidator found");
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onEnter: function(args) {
                        var hostname = new ObjC.Object(args[3]);
                        console.log("[TrustKit] Checking: " + hostname.toString());
                    },
                    onLeave: function(retval) {
                        console.log("[TrustKit] Original result: " + retval + " → forcing YES");
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit hooked with logs");
            } else {
                console.log("[!] evaluateTrust method not found");
            }
        } else {
            console.log("[!] TSKPinningValidator class not found");
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Ready — watch logs");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `


вот логи
```q
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] н
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] TrustKit force YES + proxy with LOGS...
[*] Setting up proxy...
[*] Config class: NSURLSessionConfiguration
[!] Proxy error: expected a pointer
[!] Stack: Error: expected a pointer
    at m (<input>:1)
    at <eval> (ssl-kill-switch3.js:17)
[*] Looking for TrustKit...
[*] TSKPinningValidator found
[*] TrustKit hooked with logs
[SSL Kill Switch] Ready — watch logs
[iPhone::MeetWay ]-> [TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: baket-ivaaro.storage.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES
[TrustKit] Checking: functions.yandexcloud.net
[TrustKit] Original result: 0x0 → forcing YES

```


че стало ясно:

- **TrustKit работает идеально** - форсим YES, запросы проходят
- **Прокси-ошибка** - `expected a pointer` на строке 17 (установка прокси)

`setConnectionProxyDictionary_` ожидает NSDictionary, а мы передаём JavaScript-объект

-------------

исправляем
```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit YES + fixed proxy...");

// 1. Прокси через NSDictionary
if (ObjC.available) {
    try {
        var NSDictionary = ObjC.classes.NSDictionary;
        var NSNumber = ObjC.classes.NSNumber;
        var NSString = ObjC.classes.NSString;
        
        var proxyDict = NSDictionary.dictionaryWithObjects_forKeys_([
            NSNumber.numberWithInt_(1),                    // HTTPEnable
            NSString.stringWithString_("192.168.0.100"),   // HTTPProxy
            NSNumber.numberWithInt_(8080),                 // HTTPPort
            NSNumber.numberWithInt_(1),                    // HTTPSEnable
            NSString.stringWithString_("192.168.0.100"),   // HTTPSProxy
            NSNumber.numberWithInt_(8080)                  // HTTPSPort
        ], [
            NSString.stringWithString_("HTTPEnable"),
            NSString.stringWithString_("HTTPProxy"),
            NSString.stringWithString_("HTTPPort"),
            NSString.stringWithString_("HTTPSEnable"),
            NSString.stringWithString_("HTTPSProxy"),
            NSString.stringWithString_("HTTPSPort")
        ]);
        
        console.log("[*] Proxy dict created: " + proxyDict);
        
        // Применяем к defaultSessionConfiguration
        var defaultConfig = ObjC.classes.NSURLSessionConfiguration.defaultSessionConfiguration();
        defaultConfig.setConnectionProxyDictionary_(proxyDict);
        console.log("[*] Proxy set on default config");
        
        // И к ephemeralSessionConfiguration тоже
        var ephemeralConfig = ObjC.classes.NSURLSessionConfiguration.ephemeralSessionConfiguration();
        ephemeralConfig.setConnectionProxyDictionary_(proxyDict);
        console.log("[*] Proxy set on ephemeral config");
        
    } catch(e) {
        console.log("[!] Proxy error: " + e.message);
    }
}

// 2. TrustKit форсим YES
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit forced YES");
            }
        }
    } catch(e) {}
}

console.log("[SSL Kill Switch] Ready");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `
итого, и с перехватом и без перехвата - трафик в приложении режется - ничего не работает, приложение не виснет
нужно больше логов


----------------


```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] Debug version...");

// 1. Прокси через NSDictionary
if (ObjC.available) {
    try {
        console.log("[*] Step 1: Creating NSDictionary...");
        var NSDictionary = ObjC.classes.NSDictionary;
        var NSNumber = ObjC.classes.NSNumber;
        var NSString = ObjC.classes.NSString;
        console.log("[*] Classes found: " + NSDictionary + ", " + NSNumber + ", " + NSString);
        
        var keys = [
            NSString.stringWithString_("HTTPEnable"),
            NSString.stringWithString_("HTTPProxy"),
            NSString.stringWithString_("HTTPPort"),
            NSString.stringWithString_("HTTPSEnable"),
            NSString.stringWithString_("HTTPSProxy"),
            NSString.stringWithString_("HTTPSPort")
        ];
        console.log("[*] Keys created, count: " + keys.length);
        
        var values = [
            NSNumber.numberWithInt_(1),
            NSString.stringWithString_("192.168.0.100"),
            NSNumber.numberWithInt_(8080),
            NSNumber.numberWithInt_(1),
            NSString.stringWithString_("192.168.0.100"),
            NSNumber.numberWithInt_(8080)
        ];
        console.log("[*] Values created, count: " + values.length);
        
        var proxyDict = NSDictionary.dictionaryWithObjects_forKeys_(values, keys);
        console.log("[*] Proxy dict: " + proxyDict);
        
        // Проверим, что в словаре
        console.log("[*] HTTPEnable: " + proxyDict.objectForKey_("HTTPEnable"));
        console.log("[*] HTTPProxy: " + proxyDict.objectForKey_("HTTPProxy"));
        console.log("[*] HTTPPort: " + proxyDict.objectForKey_("HTTPPort"));
        
        var defaultConfig = ObjC.classes.NSURLSessionConfiguration.defaultSessionConfiguration();
        console.log("[*] Default config before: " + defaultConfig.connectionProxyDictionary());
        
        defaultConfig.setConnectionProxyDictionary_(proxyDict);
        console.log("[*] Default config after: " + defaultConfig.connectionProxyDictionary());
        
    } catch(e) {
        console.log("[!] Proxy error at step: " + e.message);
        console.log("[!] Stack: " + e.stack);
    }
}

// 2. TrustKit без изменений
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        console.log("[TrustKit] forcing YES for request");
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit forced YES");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Ready");
ENDOFSCRIPT
```



переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

итого - тот же  и с перехватом и без перехвата - трафик в приложении режется - ничего не работает, приложение не виснет
нужно больше логов
```q
[iPhone::MeetWay ]-> %load ssl-kill-switch3.js
Are you sure you want to load a new script and discard all current state? [y/N] y
[SSL Kill Switch] Debug version...
[*] Step 1: Creating NSDictionary...
[*] Classes found: NSDictionary, NSNumber, NSString
[*] Keys created, count: 6
[*] Values created, count: 6
[!] Proxy error at step: expected a pointer
[!] Stack: Error: expected a pointer
    at m (<input>:1)
    at <eval> (ssl-kill-switch3.js:32)
[*] TrustKit forced YES
[SSL Kill Switch] Ready
[iPhone::MeetWay ]-> [TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
[TrustKit] forcing YES for request
```


-------------


на раннем этапе перехват инициализацию NSURLSession и внедрение
прокси прямо при создании

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] Init-time proxy injection...");

// 1. Перехватываем создание NSURLSessionConfiguration
if (ObjC.available) {
    try {
        var NSURLSessionConfiguration = ObjC.classes.NSURLSessionConfiguration;
        
        // Хукаем defaultSessionConfiguration
        var defaultMethod = NSURLSessionConfiguration['+ defaultSessionConfiguration'];
        if (defaultMethod) {
            Interceptor.attach(defaultMethod.implementation, {
                onLeave: function(retval) {
                    try {
                        var config = new ObjC.Object(retval);
                        console.log("[*] defaultSessionConfiguration called");
                        
                        // Создаём NSDictionary для прокси
                        var NSMutableDictionary = ObjC.classes.NSMutableDictionary;
                        var proxyDict = NSMutableDictionary.alloc().init();
                        
                        // Заполняем словарь
                        proxyDict.setObject_forKey_(1, "HTTPEnable");
                        proxyDict.setObject_forKey_("192.168.0.100", "HTTPProxy");
                        proxyDict.setObject_forKey_(8080, "HTTPPort");
                        proxyDict.setObject_forKey_(1, "HTTPSEnable");
                        proxyDict.setObject_forKey_("192.168.0.100", "HTTPSProxy");
                        proxyDict.setObject_forKey_(8080, "HTTPSPort");
                        
                        console.log("[*] Proxy dict: " + proxyDict);
                        
                        config.setConnectionProxyDictionary_(proxyDict);
                        console.log("[*] Proxy injected into default config");
                        console.log("[*] Verify: " + config.connectionProxyDictionary());
                    } catch(e) {
                        console.log("[!] Error in hook: " + e.message);
                    }
                }
            });
            console.log("[*] Hooking defaultSessionConfiguration");
        }
        
        // Хукаем ephemeralSessionConfiguration
        var ephemeralMethod = NSURLSessionConfiguration['+ ephemeralSessionConfiguration'];
        if (ephemeralMethod) {
            Interceptor.attach(ephemeralMethod.implementation, {
                onLeave: function(retval) {
                    try {
                        var config = new ObjC.Object(retval);
                        console.log("[*] ephemeralSessionConfiguration called");
                        
                        var NSMutableDictionary = ObjC.classes.NSMutableDictionary;
                        var proxyDict = NSMutableDictionary.alloc().init();
                        
                        proxyDict.setObject_forKey_(1, "HTTPEnable");
                        proxyDict.setObject_forKey_("192.168.0.100", "HTTPProxy");
                        proxyDict.setObject_forKey_(8080, "HTTPPort");
                        proxyDict.setObject_forKey_(1, "HTTPSEnable");
                        proxyDict.setObject_forKey_("192.168.0.100", "HTTPSProxy");
                        proxyDict.setObject_forKey_(8080, "HTTPSPort");
                        
                        config.setConnectionProxyDictionary_(proxyDict);
                        console.log("[*] Proxy injected into ephemeral config");
                    } catch(e) {
                        console.log("[!] Error in ephemeral hook: " + e.message);
                    }
                }
            });
            console.log("[*] Hooking ephemeralSessionConfiguration");
        }
        
    } catch(e) {
        console.log("[!] Setup error: " + e.message);
    }
}

// 2. TrustKit
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        console.log("[TrustKit] Forcing YES");
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit hooked");
            }
        }
    } catch(e) {}
}

console.log("[SSL Kill Switch] Ready — waiting for connections");
ENDOFSCRIPT
```


переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

приложеине полностью виснет


-----

логи!

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Observer] Logging ALL SSL-related calls...");

// 1. Логируем TrustKit
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            console.log("[*] TSKPinningValidator found, methods:");
            var methods = TSKPinningValidator.$ownMethods;
            methods.forEach(function(m) { console.log("  " + m); });
            
            // Логируем evaluateTrust:forHostname: — ТОЛЬКО логи, без модификации
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onEnter: function(args) {
                        var hostname = new ObjC.Object(args[3]);
                        console.log("[TrustKit] evaluateTrust FOR: " + hostname);
                    },
                    onLeave: function(retval) {
                        console.log("[TrustKit] evaluateTrust RETURNS: " + retval);
                    }
                });
            }
        }
    } catch(e) {
        console.log("[!] TrustKit: " + e.message);
    }
}

// 2. Логируем NSURLSession делегат (без модификации!)
if (ObjC.available) {
    try {
        var NSObject = ObjC.classes.NSObject;
        var challenge = NSObject['- URLSession:didReceiveChallenge:completionHandler:'];
        if (challenge) {
            Interceptor.attach(challenge.implementation, {
                onEnter: function(args) {
                    var authChallenge = new ObjC.Object(args[3]);
                    var authMethod = authChallenge.protectionSpace().authenticationMethod();
                    var host = authChallenge.protectionSpace().host();
                    console.log("[NSURLSession] Challenge for: " + host + " method: " + authMethod);
                }
            });
            console.log("[*] NSURLSession challenge watcher active");
        }
    } catch(e) {
        console.log("[!] NSURLSession: " + e.message);
    }
}

// 3. Логируем системные вызовы Security (ТОЛЬКО логи!)
try {
    var secModule = Process.findModuleByName("Security");
    if (secModule) {
        secModule.enumerateExports().forEach(function(exp) {
            if (exp.name === "_SecTrustEvaluate" || exp.name === "_SecTrustEvaluateWithError") {
                Interceptor.attach(exp.address, {
                    onEnter: function(args) {
                        console.log("[Security] " + exp.name + " called");
                    },
                    onLeave: function(retval) {
                        console.log("[Security] " + exp.name + " returns: " + retval);
                    }
                });
                console.log("[*] Watching " + exp.name);
            }
        });
    } else {
        console.log("[!] Security module not found");
    }
} catch(e) {
    console.log("[!] Security hook: " + e.message);
}

console.log("[SSL Observer] All watchers active — DO NOT MODIFY ANYTHING");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

и приложение полностью зависло!

----------




ПЛАН _ КАПКАН

```q
cat > ~/ssl-kill-switch3.js << 'ENDOFSCRIPT'
console.log("[SSL Kill Switch] TrustKit + URLProtocol bypass...");

// 1. TrustKit форсим YES
if (ObjC.available) {
    try {
        var TSKPinningValidator = ObjC.classes.TSKPinningValidator;
        if (TSKPinningValidator) {
            var evaluateTrust = TSKPinningValidator['- evaluateTrust:forHostname:'];
            if (evaluateTrust) {
                Interceptor.attach(evaluateTrust.implementation, {
                    onLeave: function(retval) {
                        retval.replace(ptr(1));
                    }
                });
                console.log("[*] TrustKit.evaluateTrust hooked");
            }
            
            // Хукаем handleChallenge — через него идёт URLProtocol
            var handleChallenge = TSKPinningValidator['- handleChallenge:completionHandler:'];
            if (handleChallenge) {
                Interceptor.replace(handleChallenge.implementation, 
                    new NativeCallback(function(self, sel, challenge, completionHandler) {
                        console.log("[*] TrustKit.handleChallenge bypassed");
                        var completion = new ObjC.Object(completionHandler);
                        // NSURLSessionAuthChallengeUseCredential = 0, передаём serverTrust
                        var authChallenge = new ObjC.Object(challenge);
                        var trust = authChallenge.protectionSpace().serverTrust();
                        var credential = ObjC.classes.NSURLCredential.credentialForTrust_(trust);
                        completion(0, credential);
                    }, 'void', ['pointer', 'pointer', 'pointer', 'pointer'])
                );
                console.log("[*] TrustKit.handleChallenge replaced");
            }
        }
    } catch(e) {
        console.log("[!] TrustKit error: " + e.message);
    }
}

console.log("[SSL Kill Switch] Ready");
ENDOFSCRIPT
```

переподключаюсь
`     frida -U MeetWay     `

подгружаю скрипт
`      %load ssl-kill-switch3.js       `

приложеине не виснет - но и трафика нет через него  !!!!!!


--------

### план № 24592465926459 - "зае-ался"

найти вызов первоначальной функции - где инициализируется эта функция которая запускает процессы пиллинга
и фридой ее заблокировать нахер!


```q
// Ищем AppDelegate через полный Swift-нейминг
var appDelegateClass = null;
for (var className in ObjC.classes) {
    if (className.toLowerCase().includes('appdelegate')) {
        console.log("[FOUND] " + className);
        appDelegateClass = ObjC.classes[className];
    }
}

if (appDelegateClass) {
    console.log("\n[*] AppDelegate methods:");
    var allMethods = appDelegateClass.$ownMethods.concat(appDelegateClass.$ownClassMethods);
    allMethods.forEach(function(m) {
        console.log("  " + m);
    });
} else {
    console.log("[!] AppDelegate not found, searching all classes with 'setup' method...");
    for (var className in ObjC.classes) {
        var cls = ObjC.classes[className];
        var methods = cls.$ownMethods.concat(cls.$ownClassMethods);
        for (var i = 0; i < methods.length; i++) {
            if (methods[i].toLowerCase().includes('trustkit') || 
                methods[i].toLowerCase().includes('grpc') ||
                methods[i].toLowerCase().includes('sslpinning')) {
                console.log("[FOUND] Class: " + className + " -> " + methods[i]);
            }
        }
    }
}
```


```q

[FOUND] GULAppDelegateSwizzler
[FOUND] GULAppDelegateObserver
[FOUND] SwiftUI.TestingAppDelegate
[FOUND] SwiftUI.AppDelegate
[FOUND] MeetWay.AppDelegate

[*] AppDelegate methods:
  - application:didFinishLaunchingWithOptions:
  - application:didRegisterForRemoteNotificationsWithDeviceToken:
  - application:didFailToRegisterForRemoteNotificationsWithError:
  - messaging:didReceiveRegistrationToken:
  - application:openURL:options:
  - userNotificationCenter:willPresentNotification:withCompletionHandler:
  - userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler:
  - init
  undefined
[iPhone::MeetWay ]->
```

и тут я решил уже посмотреть, почему я не вижу среди этих методов свою функцию : которая там точно есть!

и вот в чем дело то

в коде есть функция setupTrustKitWithGRPC
она полностью отвечает за весь sll pinnign в приложении

и если ее вырубить, не запускать - то ничего работать и не будет!

и она вызывается в методе апделегата - didFinishLaunchingWithOptions

и не видно ее при перехвате фридой - потому, что она PRIVATE!!!!
она не видна в ObjC-рантайме!!!

<img src="../../../assets/Снимок2026-04-2518.40.39.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




поэтому - мой план:
через радар2 - ищу все функции че есть в приложении, нахожу подозрительные , связанные с sllpinning
и проверяю все места в коде - где такая функция встречается, затем, я найду ее название и ее адресс в памяти

и через фриду вырублю эту функцию нахрен!

еду в бинарник
```c
r2 /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/MeetWay.debug.dyliba

[0x...]> aaa

----------------

afl~   выведу все строки, если начнет бесить
```

делаю пейлоад - список слов - которые могут в теории называтсья для функций  SSL/TLS/Pinning

```json
afl~ssl
afl~SSL
afl~tls
afl~TLS
afl~pinning
afl~Pinning
afl~pinn
afl~Pinn
afl~certificate
afl~Certificate
afl~cert
afl~Cert
afl~publicKey
afl~PublicKey
afl~spki
afl~SPKI
afl~hash
afl~Hash
afl~validate
afl~Validate
afl~verify
afl~Verify
afl~trust
afl~Trust
afl~challenge
afl~Challenge
afl~credential
afl~Credential
afl~serverTrust
afl~ServerTrust
afl~evaluate
afl~Evaluate
```


и вот первый улов
```q
0x004ed1b8    1    496 sym.MeetWay.AppDelegate.showSSLPinningAlert.allocator_...F_

0x004ec554    1   3172 sym.MeetWay.AppDelegate.setupTrustKitWithGRPC.allocator.

```
и вот она функция в appdelegate
setupTrustKitWithGRPC (выглядит подозрительно, так как TrustKit - это как раз лайбра для пиннинга sll)
по адрессу  0x004ec554

<img src="../../../assets/Снимок2026-04-2519.03.23.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




------

теперь нужно понять, в каких метах в коде вызывается функция эта
0x004ec554    1   3172 sym.MeetWay.AppDelegate.setupTrustKitWithGRPC.allocator.

```q
[0x004ec554]> axt @0x004ec554
sym.MeetWay.AppDelegate.application.allocator.didFinishLaunchingWithOptions...SbSo13UIApplicationC_SDySo0k6LaunchJ3KeyaypGSgtF 0x4eb9dc [CALL:--x] bl sym.MeetWay.AppDelegate.setupTrustKitWithGRPC.allocator.
```
эта функция вызывается в AppDelegate.application.allocator.didFinishLaunchingWithOptions
(но это и так было видно)

ладно, попробую фридой просто поймать и заблокировать вызов этой функции (0x004ec554 setupTrustKitWithGRPC)еще до запуска приложения


```js
cat > ~/block-setup-ssl.js << 'ENDOFSCRIPT'
console.log("[BLOCK] Targeting setupTrustKitWithGRPC...");

// Ждём загрузки модуля
var module = Process.findModuleByName("MeetWay.debug.dylib");
if (!module) {
    console.log("[!] Module not found yet, waiting...");
    // Пробуем найти позже
    var interval = setInterval(function() {
        module = Process.findModuleByName("MeetWay.debug.dylib");
        if (module) {
            clearInterval(interval);
            hookFunction(module);
        }
    }, 100);
} else {
    hookFunction(module);
}

function hookFunction(module) {
    console.log("[*] Module base: " + module.base);
    
    // setupTrustKitWithGRPC оффсет
    var setupPinningOffset = 0x4ec554;
    var setupPinningAddr = module.base.add(setupPinningOffset);
    
    console.log("[*] setupTrustKitWithGRPC at: " + setupPinningAddr);
    
    // БЛОКИРУЕМ функцию полностью
    Interceptor.replace(setupPinningAddr, new NativeCallback(function() {
        console.log("[BLOCK] setupTrustKitWithGRPC CALLED AND BLOCKED!");
        console.log("[BLOCK] SSL Pinning DEAD — returning immediately");
        // Ничего не делаем, просто выходим
        return; // void функция
    }, 'void', []));
    
    console.log("[BLOCK] setupTrustKitWithGRPC REPLACED with empty function");
    console.log("[BLOCK] SSL Pinning should be completely disabled!");
}

console.log("[BLOCK] Script loaded — waiting for module...");
ENDOFSCRIPT
```

нужно запустить приложение так, чтобы фрида изначально запускала скрипт и только потом запуск приложения происходил, чтобы функция не успела выполниться!


```q
узнал  bundle-id
evgeniy@Evgeniys-MacBook-Pro ~ % frida-ps -Uai | grep MeetWay
2278  MeetWay                  AIVARO22-2025-1.0

и потом:
запуск

frida -U -f AIVARO22-2025-1.0 -l ~/block-setup-ssl.js
```

АХАХАХХАХА!!!! УРА БЛЕАТ!!!!!

ПОЛУЧИЛОСЬ!!!

ААААААА!!!!!


УХАХАХАХ!!!

СУТКИ УБИЛ НА УСТАНОВКУ ПИННИНГА И ЕГО ПОСЛЕДУЮЩИЙ ВЗЛОМ!!!
 ВСЕ ПОЛУЧИЛОСЬ!!!!
 
 Я ОБОШЕЛ SLL PINNING С ТРОЙНОЙ ЗАЩИТОЙ (но у меня был расшифрованный IPA) , я не подглядывал в него, то есть без подсказок!
 
 ТРАФИК СПОКОЙНО ЛЬЕТСЯ ЧЕРЕЗ BURP !!!! СКРИПТ ФРИДЫ ВЫРУБИЛ ВСЕ ПРОВЕРКИ БЛАГОДАРЯ ОТКЛЮЧЕНИЮ ВЫЗОВА ФУНКЦИИ  КОТОРАЯ ИНИЦИАЛИЗИРОВАЛА В ПРИЛОЖЕНИИ SLL PINNING!
 
 
вот - красота! айфон без джейлбрейка, (айфон11 ios 26) включен перехват через burp - и приложение спокойно доверяет его сертификату, хотя, там реализована тройная защита!! ее удалось обойти банальным методом, через радар2 в бинарнике нашел  название функции перехвата - и оно имело понятное имя - которое говорило само за себя   `setupTrustKitWithGRPC` , а далее фридой просто отключил вызов этой функции до запуска приложения!
и все, приложение начало работать без sll pinning

вот скрин (на нем, включен перехват через бурп, при этом - с помощью фрида вырублен весь SLL Pinning в приложении)

ЭТО победа!

<img src="../../../assets/Снимок2026-04-2519.35.13.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




-------

### как можно было бы усилить защиту:

во первых - ОБФУСЦИРОВАТЬ имена функций отвечающих за SLL Pinning
(так бы я не сог прямо и быстрой найти конкретную функцию!)

во вторых - завязать функцию эту так, чтобы напрмиер, в ее вызове была какая-то другая системная функция - необходимая для работы приложения, чтобы при отключении функции SLL - приложение падало вместе с функцией (или задать в коде проверки на работу функции, типо если логов с работы функции нет - то приложение должно упасть или предупредить юзера)

в третьих - расскидать логику работы функции на несколько вызовов (функций)



но так или иначе, если очень долго капаться - то можно было бы и это обойти, но на это ушло бы очень много времени!!!

----
# вывод 

победная тактика была такой:

1) обнаружил, что есть sll pinning
2) через radar2 нашел функцию которая отвечает за sllpinning
3) отключил вызов данной функции с помощью фрида

(МОЖНО БЫЛО БЫ ИСКАТЬ НУЖНУЮ ФУНКЦИЮ И СПОСОБОМ, СОБИРАЯ В РАНТАЙМЕ ВСЕ ФУНКЦИИ, (ЭТО ЕСЛИ ЕСТЬ ОБФУСКАЦИЯ) И ПООЧЕРЕДНО ОТКЛЮЧАТЬ ИХ - В ДАННОМ СЛУЧАЕ - ЭТО БЫ СРАБОТАЛО)

-----------
