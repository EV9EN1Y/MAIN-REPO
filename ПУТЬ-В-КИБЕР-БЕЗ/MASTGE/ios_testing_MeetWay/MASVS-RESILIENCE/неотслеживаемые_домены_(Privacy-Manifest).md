# MASTG-TEST-0281
Undeclared Known Tracking Domains

--------

Начиная с iOS 17, Apple требует, чтобы приложения, которые собирают данные о пользователе для отслеживания (tracking), содержали специальный **«Манифест конфиденциальности» (Privacy Manifest)**. В этом файле (`PrivacyInfo.xcprivacy`) разработчик **обязан перечислить все домены**, через которые происходит трекинг

----------
суть теста в том, чтобы просто проверить, указаны ли все домены в Privacy Manifest    в (`NSPrivacyTrackingDomains`), которые использует приложение для трекинга 

-----

риски, если что-либо не указано:
```c 

🚦 Юридические и 🚦 репутационные риски: Нарушение правил App Store (приложение могут отклонить или удалить) и требований законов о конфиденциальности (GDPR в Европе, CCPA в Калифорнии и т.д.). 
Это грозит штрафами и блокировкой приложения
   
🚦 Нарушение прав пользователя: Приложение собирает данные для отслеживания без явного согласия пользователя, что подрывает доверие
  
🚦 Утечка данных: Трекинговые домены могут быть связаны с передачей идентификаторов устройства (IDFV, IDFA) или других данных на сторонние серверы, которые не контролируются разработчиком. Это увеличивает поверхность для атаки
```

--------

тестирование выполняется статическим анализом (поиск в коде и манифестах) и динамическим перехватом трафика с его анализом

-------

вот здесь собраны сотни популярных доменов, то есть это можно использовать, чтобы по ним выполнять поиск в бинарнике

```http
-
Exodus API (JSON): https://reports.exodus-privacy.eu.org/api/trackers
   
Exodus ETIP (платформа): https://etip.exodus-privacy.eu.org
```

---------



начинаю выполнять тестирование

тестирование на приложении MeetWay
👉 (о приложении) [[0_MeetWay]]

##### 🟣 буду проводить анализ бинарника

бинарник кинул на рабочий стол
перехожу в папку 

`cd ~/Desktop/MeetWay.app/`

ищу файл манифест

`find . -name "PrivacyInfo.xcprivacy"`

результаты поиска

```http
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % find . -name "PrivacyInfo.xcprivacy"
----------------------------------

./Frameworks/grpcpp.framework/grpcpp.bundle/PrivacyInfo.xcprivacy
./Frameworks/GoogleDataTransport.framework/GoogleDataTransport_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseFirestore.framework/FirebaseFirestore_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/GTMSessionFetcher.framework/GTMSessionFetcher_Core_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FBLPromises.framework/FBLPromises_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/AWSS3.framework/AWSS3.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseCoreInternal.framework/FirebaseCoreInternal_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseCore.framework/FirebaseCore_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/GoogleUtilities.framework/GoogleUtilities_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseAuth.framework/FirebaseAuth_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/AWSCore.framework/AWSCore.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseMessaging.framework/FirebaseMessaging_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseCoreExtension.framework/FirebaseCoreExtension_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseFirestoreInternal.framework/FirebaseFirestoreInternal_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/nanopb.framework/nanopb_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/leveldb.framework/leveldb_Privacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/grpc.framework/grpc.bundle/PrivacyInfo.xcprivacy
./Frameworks/openssl_grpc.framework/openssl_grpc.bundle/PrivacyInfo.xcprivacy
./Frameworks/absl.framework/xcprivacy.bundle/PrivacyInfo.xcprivacy
./Frameworks/FirebaseInstallations.framework/FirebaseInstallations_Privacy.bundle/PrivacyInfo.xcprivacy

```

в общем то - все что нашлось - только манифесты от сторонних библиотек

и файла PrivacyInfo.xcprivacy - не могу найти

**И отсутствие манифеста у самого приложения** это - - - > нарушение правил App Store (с 1 мая 2024). Apple требует, чтобы _каждое_ приложение имело свой `PrivacyInfo.xcprivacy` в корне `.app`

вот так , посмотреть содержимое каждого подозрительного пути 

```q
plutil -p ./MeetWay.app/./Frameworks/grpcpp.framework/grpcpp.bundle/PrivacyInfo.xcprivacy
```

```q
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la
total 124904
-rwxr-xr-x   1 evgeniy  staff     35024  9 апр 10:07 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96  7 апр 00:34 _CodeSignature
drwxr-xr-x@ 17 evgeniy  staff       544 14 апр 13:51 .
drwx------@ 43 evgeniy  staff      1376 14 апр 15:58 ..
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

не вижу здесь манифеста никакого 
и так как в корне `MeetWay.app` нет файла `PrivacyInfo.xcprivacy`. Это уже нарушение требований Apple

ТО ЕСТЬ - ТЕСТ ПРОВАЛЕН!

-------
теперь нужно понять , какие домены вообще использует приложение, чтобы указать их в манифесте

и тут нужно использовать несколько подходов

1) burp  - перехватить трафик и собрать все домены, но это не гарантирует, что получиться собрать все домены, так как могут быть скрытые!
2) через strigs чекнуть список доменов и сигнатур любых возможных доменов

```q
strings -a -8 MeetWay.debug.dylib | grep -E -i "(analytics|tracking|crashlytics|firebase|google|facebook|appsflyer|adjust|amplitude|mixpanel|yandex|metric|doubleclick|googletagmanager|app-measurement|exodus)" | sort -u

---------

find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -E -o "https?://[a-zA-Z0-9./?=_-]+" && strings -a -8 "$0" 2>/dev/null | grep -E -o "[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(/|:|\")"' {} \; | sort -u


без дублей

find . -type f -exec strings -a -8 {} \; 2>/dev/null | grep -E -o "[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" | sort -u

----------------------

ЛИБО ВОТ ТАК ИСКАТЬ СРАЗУ С УКАЗАНИЕМ КОНКРЕТНЫХ ФАЙЛОВ

find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -E -i "(analytics|tracking|crashlytics|firebase|google|facebook|appsflyer|adjust|amplitude|mixpanel|yandex|metric|doubleclick|googletagmanager|app-measurement|exodus)" | head -1 | xargs -I {} echo "В $0: {}"' {} \;



find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -E -o "https?://[a-zA-Z0-9./?=_-]+" | head -1 | xargs -I {} echo "В $0: {}"' {} \; && find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -E -o "[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" | head -1 | xargs -I {} echo "В $0: {}"' {} \;


find . -type f -exec sh -c 'strings -a -8 "$0" 2>/dev/null | grep -E -o "[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" | head -1 | xargs -I {} echo "В $0: {}"' {} \; | sort -u
```

можно использовать списки из 
```http
-
Exodus API (JSON): https://reports.exodus-privacy.eu.org/api/trackers
   
Exodus ETIP (платформа): https://etip.exodus-privacy.eu.org
```


результаты поиска отфильтровал на уникальные значения

и нужно понять, где и что из этого используется для треккинга!


----------

результаты теста

```c
первый тест      (URL и домены)

В http://docs.aws.amazon.com/mobile/sdkforios/developerguide/setup.html: http://docs.aws.amazon.com/mobile/sdkforios/developerguide/setup.html
В http://example.invalid: http://example.invalid
В http://mozilla.org/MPL/2.0/.: http://mozilla.org/MPL/2.0/.
В http://www.apple.com/DTDs/PropertyList-1.0.dtd: http://www.apple.com/DTDs/PropertyList-1.0.dtd
В http://www.google.com/: http://www.google.com/
В http://www.iec.ch: http://www.iec.ch
В https://console.firebase.google.com/: https://console.firebase.google.com/
В https://dummyapiverylong-dummy.google.com/dummy/api/very/long: https://dummyapiverylong-dummy.google.com/dummy/api/very/long
В https://firebase.google.com/docs/cloud-messaging/ios/client: https://firebase.google.com/docs/cloud-messaging/ios/client
В https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg: https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg
В https://www.googleapis.com/auth/cloud-platform: https://www.googleapis.com/auth/cloud-platform



второй тест      (домены без протокола)

В 3465d3a7-cef0-4d47-874c-c69cff47ab81.jpg: 3465d3a7-cef0-4d47-874c-c69cff47ab81.jpg
В accesslog.proto: accesslog.proto
В AIVARO22-2025-1.0.background: AIVARO22-2025-1.0.background
В AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEMY-2e.AAIVARen: AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEMY-2e.AAIVARen
В arena.cc: arena.cc
В arm64-apple-ios.swiftinterface: arm64-apple-ios.swiftinterface
В ascii.cc: ascii.cc
В AsyncAwait.swift: AsyncAwait.swift
В com.amazonaws.AWSS: com.amazonaws.AWSS
В com.apple.Previews.StubExecutor: com.apple.Previews.StubExecutor
В com.firebase.core: com.firebase.core
В com.google.GDTCCTUploader: com.google.GDTCCTUploader
В com.google.GTMSessionFetcher: com.google.GTMSessionFetcher
В currentItem.status: currentItem.status
В FirebaseApp.name: FirebaseApp.name
В FirebaseAuth.ActionCodeInfo: FirebaseAuth.ActionCodeInfo
В grpc.health.v1.Health: grpc.health.v1.Health
В HeartbeatController.swift: HeartbeatController.swift
В Info.plist: Info.plist
В mozilla.org: mozilla.org
В org.cocoapods.abslS: org.cocoapods.abslS
В org.cocoapods.AWSCoreS: org.cocoapods.AWSCoreS
В org.cocoapods.AWSS: org.cocoapods.AWSS
В org.cocoapods.FBLPromises: org.cocoapods.FBLPromises
В org.cocoapods.FBLPromisesS: org.cocoapods.FBLPromisesS
В org.cocoapods.FirebaseAppCheckInteropS: org.cocoapods.FirebaseAppCheckInteropS
В org.cocoapods.FirebaseAuth: org.cocoapods.FirebaseAuth
В org.cocoapods.FirebaseAuthInteropS: org.cocoapods.FirebaseAuthInteropS
В org.cocoapods.FirebaseAuthS: org.cocoapods.FirebaseAuthS
В org.cocoapods.FirebaseCore: org.cocoapods.FirebaseCore
В org.cocoapods.FirebaseCoreExtension: org.cocoapods.FirebaseCoreExtension
В org.cocoapods.FirebaseCoreExtensionS: org.cocoapods.FirebaseCoreExtensionS
В org.cocoapods.FirebaseCoreInternal: org.cocoapods.FirebaseCoreInternal
В org.cocoapods.FirebaseCoreInternalS: org.cocoapods.FirebaseCoreInternalS
В org.cocoapods.FirebaseCoreS: org.cocoapods.FirebaseCoreS
В org.cocoapods.FirebaseFirestore: org.cocoapods.FirebaseFirestore
В org.cocoapods.FirebaseFirestoreInternal: org.cocoapods.FirebaseFirestoreInternal
В org.cocoapods.FirebaseFirestoreInternalS: org.cocoapods.FirebaseFirestoreInternalS
В org.cocoapods.FirebaseFirestoreS: org.cocoapods.FirebaseFirestoreS
В org.cocoapods.FirebaseInstallations: org.cocoapods.FirebaseInstallations
В org.cocoapods.FirebaseInstallationsS: org.cocoapods.FirebaseInstallationsS
В org.cocoapods.FirebaseMessaging: org.cocoapods.FirebaseMessaging
В org.cocoapods.FirebaseMessagingS: org.cocoapods.FirebaseMessagingS
В org.cocoapods.FirebaseSharedSwiftS: org.cocoapods.FirebaseSharedSwiftS
В org.cocoapods.GoogleDataTransport: org.cocoapods.GoogleDataTransport
В org.cocoapods.GoogleDataTransportS: org.cocoapods.GoogleDataTransportS
В org.cocoapods.GoogleUtilities: org.cocoapods.GoogleUtilities
В org.cocoapods.GoogleUtilitiesS: org.cocoapods.GoogleUtilitiesS
В org.cocoapods.gRPCCertificates: org.cocoapods.gRPCCertificates
В org.cocoapods.grpcppS: org.cocoapods.grpcppS
В org.cocoapods.grpcS: org.cocoapods.grpcS
В org.cocoapods.GTMSessionFetcher: org.cocoapods.GTMSessionFetcher
В org.cocoapods.GTMSessionFetcherS: org.cocoapods.GTMSessionFetcherS
В org.cocoapods.leveldb: org.cocoapods.leveldb
В org.cocoapods.leveldbS: org.cocoapods.leveldbS
В org.cocoapods.nanopb: org.cocoapods.nanopb
В org.cocoapods.nanopbS: org.cocoapods.nanopbS
В org.cocoapods.openssl: org.cocoapods.openssl
В org.cocoapods.RecaptchaInteropS: org.cocoapods.RecaptchaInteropS
В org.cocoapods.xcprivacyS: org.cocoapods.xcprivacyS
В org.cocoapods.YandexLoginSDKS: org.cocoapods.YandexLoginSDKS
В promise.isPending: promise.isPending
В ss.SSS: ss.SSS
В u.base.can: u.base.can
В www.apple.com: www.apple.com
В www.google.com: www.google.com



третий тест    (аналитика/трекинг)

В <key>FirebaseAppCheckInterop.framework/FirebaseAppCheckInterop</key>: <key>Frameworks/FirebaseAppCheckInterop.framework/FirebaseAppCheckInterop</key>
В <key>FirebaseAuth_Privacy.bundle/Info.plist</key>: <key>FirebaseAuth_Privacy.bundle/Info.plist</key>
В <key>FirebaseCore_Privacy.bundle/Info.plist</key>: <key>FirebaseCore_Privacy.bundle/Info.plist</key>
В <key>FirebaseCoreExtension_Privacy.bundle/Info.plist</key>: <key>FirebaseCoreExtension_Privacy.bundle/Info.plist</key>
В <key>FirebaseCoreInternal_Privacy.bundle/Info.plist</key>: <key>FirebaseCoreInternal_Privacy.bundle/Info.plist</key>
В <key>FirebaseFirestore_Privacy.bundle/Info.plist</key>: <key>FirebaseFirestore_Privacy.bundle/Info.plist</key>
В <key>FirebaseFirestoreInternal_Privacy.bundle/Info.plist</key>: <key>FirebaseFirestoreInternal_Privacy.bundle/Info.plist</key>
В <key>FirebaseInstallations_Privacy.bundle/Info.plist</key>: <key>FirebaseInstallations_Privacy.bundle/Info.plist</key>
В <key>FirebaseMessaging_Privacy.bundle/Info.plist</key>: <key>FirebaseMessaging_Privacy.bundle/Info.plist</key>
В <key>GoogleDataTransport_Privacy.bundle/Info.plist</key>: <key>GoogleDataTransport_Privacy.bundle/Info.plist</key>
В <key>GoogleUtilities_Privacy.bundle/Info.plist</key>: <key>GoogleUtilities_Privacy.bundle/Info.plist</key>
В <key>NSPrivacyCollectedDataTypeTracking</key>: <key>NSPrivacyCollectedDataTypeTracking</key>
В <key>NSPrivacyTracking</key>: <key>NSPrivacyTracking</key>
В <string>YandexLoginSDK</string>: <string>YandexLoginSDK</string>
В AWSS3AnalyticsAndOperator: AWSS3AnalyticsAndOperator
В biometricInfo: biometricInfo
В com.google.FBLPromises.Await: com.google.FBLPromises.Await
В com.google.GTMSessionFetcher: com.google.GTMSessionFetcher
В FireBaseManager: FireBaseManager
В FirebaseAppDelegateProxyEnabled_: FirebaseAppDelegateProxyEnabled_
В FirebaseAppDelegateProxyEnabled]GCM_SENDER_ID]GOOGLE_APP_ID^IS_ADS_ENABLED_: FirebaseAppDelegateProxyEnabled]GCM_SENDER_ID]GOOGLE_APP_ID^IS_ADS_ENABLED_
В google/protobuf/any.proto: google/protobuf/any.proto
В graph.facebook.com: graph.facebook.com
В # Issuer: CN=GTS Root R1 O=Google Trust Services LLC: # Issuer: CN=GTS Root R1 O=Google Trust Services LLC
В N4grpc18BackendMetricStateE: N4grpc18BackendMetricStateE
В UIRequiredDeviceCapabilitiesV24G624RenFirebaseAuth_: UIRequiredDeviceCapabilitiesV24G624RenFirebaseAuth_
В UIRequiredDeviceCapabilitiesV24G624RenFirebaseCore_: UIRequiredDeviceCapabilitiesV24G624RenFirebaseCore_
```


в итоге можно сказать, что здесь почти все - это просто вшитые в код библиотек сыылки. то есть базовый набор фрейм-ворков

и окончательно понять, с чем взаимодействует приложение - можно через динамический анализ!

поэтому подрубаю burp!!!!

перенаправляю через общий wifi трафик с телефона на комп!

потыкал приложение везде

перехватил трафик приложения, и вот уникальные энд поинты

```http
УНИКАЛЬНЫЕ ДОМЕНЫ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. login.yandex.ru          - авторизация
2. functions.yandexcloud.net  - там все рабочие методы приложения
3. baket-ivaaro.storage.yandexcloud.net   - одно из хранилищ приложения
4. captive.apple.com      -   системный

УНИКАЛЬНЫЕ ЭНДПОИНТЫ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• GET /info?format=json                    (login.yandex.ru)
• POST /d4efavboiige7c7leqkf               (functions.yandexcloud.net)
• GET /users/{user_id}/{file_name}.jpg     (storage.yandexcloud.net)
• GET /                                     (captive.apple.com)

УНИКАЛЬНЫЕ HTTP МЕТОДЫ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• GET  (3 эндпоинта)
• POST (1 эндпоинт)


ИТОГО 
 
	в трафике НЕТ обнаружения классических tracking доменов
	
- facebook.com
- google analytics  
- firebase analytics
- appsflyer
- adjust
- mixpanel
```

итог:

нужно создать файл в корневой директории MeetWay.app
с названием `PrivacyInfo.xcprivacy` 
и в нем указать, что приложение НЕ занимается трекингом, так для трекинга данное приложение ничего не использует!

но тест все равно провален, так как файл PrivacyInfo.xcprivacy отсутствовал вообще!


вот так должен примерно выглядеть итоговый файл PrivacyInfo.xcprivacy в корневой директории MeetWay.app

```q
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" 
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- КЛЮЧЕВОЙ МОМЕНТ: NSPrivacyTracking = false -->
    <key>NSPrivacyTracking</key>
    <false/>
    
    <!-- Если трекинга нет, массив доменов пустой -->
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    
    <!-- Данные, которые собирает приложение (если собирает) -->
    <key>NSPrivacyCollectedDataTypes</key>
    <array>
        <!-- Например, email для авторизации -->
        <dict>
            <key>NSPrivacyCollectedDataType</key>
            <string>NSPrivacyCollectedDataTypeEmail</string>
            <key>NSPrivacyCollectedDataTypeLinked</key>
            <true/>
            <key>NSPrivacyCollectedDataTypeTracking</key>
            <false/>
            <key>NSPrivacyCollectedDataTypePurposes</key>
            <array>
                <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```