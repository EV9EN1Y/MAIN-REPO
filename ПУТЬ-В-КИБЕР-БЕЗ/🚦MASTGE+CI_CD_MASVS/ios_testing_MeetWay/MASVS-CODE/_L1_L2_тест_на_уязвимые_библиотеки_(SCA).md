# MASTG-TEST-0273: 
выявление зависимостей с известными уязвимостями путем сканирования артефактов менеджеров зависимостей


если в проекте используются сторонние библиотеки, фрейм-ворки
(CocoaPods, Swift Package Manager, Carthage, или просто ручные библиотеки)

то нужно проверить их, для начала, на известные уязвимости в них!

-----

1 - узнать сами библиотеки
2 - узнать версии каждой библиотеки
3 - свериться, есть ли уязвимости для текущих версий библиотек

(SCA - Software Composition Analysis) - это какой-нибудь спец анализатор, который проверяет , есть ли уязвимости для текущих версий библиотек, например OWASP Dependency-Check

------

есть два путя...

🟣 первый путь (простой) - спросить у разработчиков файлы манифесты (`Podfile.lock`, `Package.resolved`, `Cartfile.resolved`) и прогнать их через `OWASP Dependency-Check`, `Snyk`, `Trivy` или даже в `npm audit` (если там React Native)

🟣 второй путь  - это искать все это через бинарник, (через `strings`, `nm`, `otool -L`, или реверс в Ghidra), ну и гуглить или через сканер прогнать

------

итак - путь через бинарник (по конкретным папкам)

```bash
strings ~/Desktop/MeetWay.app/MeetWay | grep -iE "alamofire|afnetworking|firebase|google|realm|swiftyjson|sdwebimage|moya|rxswift|snapkit|lottie|kingfisher|charts|hero|vibrancy|fakery|cocoapods|version|\.podspec|package|license|copyright|dependencies|AFNetworking|Alamofire|Firebase|Realm|SwiftyJSON|SDWebImage|Moya|RxSwift|SnapKit|Lottie|Kingfisher|Charts|Hero"

или так

r2 -qc "/ [A-Za-z]*(alamofire|afnetworking|firebase|google|realm|swiftyjson|sdwebimage|moya|rxswift|snapkit|lottie|kingfisher|charts|hero)" ~/Desktop/MeetWay.app/MeetWay


---------
универсал маркеры


strings ~/Desktop/MeetWay.app/MeetWay | grep -iE "copyright|license|written by|developed by|library|framework|module|initialization|static class|implementation|@interface|@implementation|https?://|www\." | sort -u

r2 -qc "izz~copyright,license,library,framework,https://,www\.,written,developed" ~/Desktop/MeetWay.app/MeetWay
```

------


----------

приступаю к тесту

тестирование на приложении MeetWay
👉 (о приложении) [[0_MeetWay]]

Так как подписки Apple Developer у меня сейчас нет, поэтому я просто через Xcode установил приложение на телефон потыкал его, и теперь в папке Delivery Data я могу найти файлы моего приложения здесь

~/Library/Developer/Xcode/DerivedData/

 И там нахожу в билдах моего приложения нужного бинарник он будет называться также как название приложения с расширением.app, ну и весить много, а внутри бинарщина

-------
Перекинул на рабочий стол бинарник MeetWay.app

открываю папку
```q
cd /Users/evgeniy/Desktop/MeetWay.app

и смотрю, че там по файлам внутри есть (для инфы, чтобы понять, что попал по адрессу)
```

для понимания, где нахожусь и че внутри!

```c
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la
total 124904
-rwxr-xr-x   1 evgeniy  staff     35024  9 апр 10:07 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96  7 апр 00:34 _CodeSignature
drwxr-xr-x@ 17 evgeniy  staff       544 14 апр 13:51 .
drwx------@ 39 evgeniy  staff      1248 14 апр 22:10 ..
-rw-r--r--@  1 evgeniy  staff         0 11 апр 10:39 aaaa
-rw-r--r--   1 evgeniy  staff     14412  7 апр 01:18 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132  7 апр 01:18 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696  7 апр 01:19 Assets.car
-rw-r--r--   1 evgeniy  staff     15136  7 апр 00:25 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864  7 апр 00:34 Frameworks
-rw-r--r--   1 evgeniy  staff       996  7 апр 00:25 GoogleService-Info.plist
-rw-r--r--@  1 evgeniy  staff      5458 14 апр 13:51 Info_readable.plist
-rw-r--r--   1 evgeniy  staff      3717  7 апр 00:26 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664  9 апр 10:07 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41070784  9 апр 10:07 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8  7 апр 00:26 PkgInfo
-rw-r--r--   1 evgeniy  staff     49852  7 апр 00:25 words.txt
```

--------

еду в папку фреймворк

```bash
ls ~/Desktop/MeetWay.app/Frameworks
```

результат
```c
absl.framework				
FirebaseMessaging.framework
AWSCore.framework			
FirebaseSharedSwift.framework
AWSS3.framework				
GoogleDataTransport.framework
FBLPromises.framework			
GoogleUtilities.framework
FirebaseAppCheckInterop.framework	
grpc.framework
FirebaseAuth.framework			
grpcpp.framework
FirebaseAuthInterop.framework		
GTMSessionFetcher.framework
FirebaseCore.framework			
leveldb.framework
FirebaseCoreExtension.framework		
nanopb.framework
FirebaseCoreInternal.framework		
openssl_grpc.framework
FirebaseFirestore.framework		
RecaptchaInterop.framework
FirebaseFirestoreInternal.framework	
YandexLoginSDK.framework
FirebaseInstallations.framework
```

далее нужно проверить их версии

```bash
plutil -p ~/Desktop/MeetWay.app/Frameworks/FirebaseCore.framework/Info.plist | grep CFBundleShortVersionString

----
или все разом

for f in absl AWSCore AWSS3 FBLPromises FirebaseAppCheckInterop FirebaseAuth FirebaseAuthInterop FirebaseCore FirebaseCoreExtension FirebaseCoreInternal FirebaseFirestore FirebaseFirestoreInternal FirebaseInstallations FirebaseMessaging FirebaseSharedSwift GoogleDataTransport GoogleUtilities grpc grpcpp GTMSessionFetcher leveldb nanopb openssl_grpc RecaptchaInterop YandexLoginSDK; do echo "=== $f ==="; plutil -p ~/Desktop/MeetWay.app/Frameworks/$f.framework/Info.plist 2>/dev/null | grep CFBundleShortVersionString; done
```

результат

```q
=== absl ===
  "CFBundleShortVersionString" => "1.20240722.0"
=== AWSCore ===
  "CFBundleShortVersionString" => "2.41.0"
=== AWSS3 ===
  "CFBundleShortVersionString" => "2.41.0"
=== FBLPromises ===
  "CFBundleShortVersionString" => "2.4.0"
=== FirebaseAppCheckInterop ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseAuth ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseAuthInterop ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseCore ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseCoreExtension ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseCoreInternal ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseFirestore ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseFirestoreInternal ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseInstallations ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseMessaging ===
  "CFBundleShortVersionString" => "11.15.0"
=== FirebaseSharedSwift ===
  "CFBundleShortVersionString" => "11.15.0"
=== GoogleDataTransport ===
  "CFBundleShortVersionString" => "10.1.0"
=== GoogleUtilities ===
  "CFBundleShortVersionString" => "8.1.0"
=== grpc ===
  "CFBundleShortVersionString" => "1.69.0"
=== grpcpp ===
  "CFBundleShortVersionString" => "1.69.0"
=== GTMSessionFetcher ===
  "CFBundleShortVersionString" => "4.5.0"
=== leveldb ===
  "CFBundleShortVersionString" => "1.22.6"
=== nanopb ===
  "CFBundleShortVersionString" => "3.30910.0"
=== openssl_grpc ===
  "CFBundleShortVersionString" => "0.0.37"
=== RecaptchaInterop ===
  "CFBundleShortVersionString" => "101.0.0"
=== YandexLoginSDK ===
  "CFBundleShortVersionString" => "3.0.2"
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```

теперь нужно прогнать их через анализатор, или прогуглить

### буду использовать OWASP Dependency-Check

но для начала.... нужно получить ключ от базы cve

##### Шаг 1 
иду сюда  https://nvd.nist.gov/developers/request-an-api-key
нужно зарегаться и получить ключ

ключ апи есть!

##### Шаг 2
 нужно поставить  Dependency-Check
https://github.com/dependency-check/DependencyCheck

скачал последний релиз и пока что на рабочий стол закинул эту папку


##### Шаг 3 
например, на раб столе создать файл с найденными библиотеками, в таком формате:

```c
cat > ~/Desktop/Podfile.lock << 'EOF'
PODS:
  - absl (1.20240722.0)
  - AWSCore (2.41.0)
  - AWSS3 (2.41.0)
  - FBLPromises (2.4.0)
  - Firebase/Core (11.15.0)
  - FirebaseFirestore (11.15.0)
  - GoogleDataTransport (10.1.0)
  - GoogleUtilities (8.1.0)
  - gRPC (1.69.0)
  - leveldb (1.22.6)
  - nanopb (3.30910.0)
  - openssl_grpc (0.0.37)
  - YandexLoginSDK (3.0.2)

DEPENDENCIES:
  - absl
  - AWSCore
  - AWSS3
  - FBLPromises
  - Firebase/Core
  - FirebaseFirestore
  - GoogleDataTransport
  - GoogleUtilities
  - gRPC
  - leveldb
  - nanopb
  - openssl_grpc
  - YandexLoginSDK
EOF
```

##### Шаг 4 
запуск сканирования!!!

```bash
  ~/Desktop/dependency-check/bin/dependency-check.sh \
  --enableExperimental \
  --nvdApiKey 9c680c05-ddee-426d-b7e3-14d7be35d1c6 \
  --scan ~/Desktop/Podfile.lock \
  --format HTML \
  --out ~/Desktop/SCA_Report
```

НО НИКУЯ НЕ РАБОТАЕТ ЭТОТ ВАШ dependency-check 
больше 2 часов убил на него!! (санкции...)

-----

план Б

иду сюда 

https://nvd.nist.gov/vuln/search
или сюда 
https://cve.circl.lu

и ручками вбиваю все свои библиотеки с версиями!


результаты проверки уязвимостей в виде обычного списка:

**absl (1.20240722.0)** - ✅ Безопасна. Нет публичных CVE для iOS.

**AWSCore / AWSS3 (2.41.0)** - ✅ Безопасны. Нет критических CVE.

**FBLPromises (2.4.0)** -✅ Безопасна. Нет известных CVE.

**Firebase (все модули, 11.15.0)** - ✅ Безопасна. Нет критических CVE.

**GoogleDataTransport (10.1.0)** - ✅ Безопасна. Нет публичных CVE.

**GoogleUtilities (8.1.0)** - ✅ Безопасна. Нет публичных CVE.

**gRPC / gRPCpp (1.69.0)** - ⚠️ Есть . Уязвимость CVE-2021-0341 (High) для grpc-okhttp (касается Android, не iOS).

**GTMSessionFetcher (4.5.0)** - ✅ Безопасна. Нет критических CVE.

**leveldb (1.22.6)** - ✅ Безопасна. Нет публичных CVE.

**nanopb (3.30910.0)** - ⚠️ Есть. CVE-2024-53984 (DoS), но это низкоуровень

**openssl_grpc (0.0.37)** - ✅ Безопасна. Нет критических CVE.

**RecaptchaInterop (101.0.0)** - ✅ Безопасна. Нет публичных CVE.

**YandexLoginSDK (3.0.2)** - ✅ Безопасна. Нет известных CVE. НО ЭТО НЕ ТОЧНО, И ВООБЩЕ _ ГДЕ ЕСТЬ ПОДОЗРЕНИЯ _ ЛУЧШЕ ПРОСТО ПОСТАВИТЬ ПОСЛЕДНЮЮ ВЕРСИЮ


------
