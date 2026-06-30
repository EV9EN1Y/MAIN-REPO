https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0225/
#### Usage of Insecure APK Signature Key Size

это тест который проверяет размер ключа которым подписано приложение
если он небольшой - то его есть шанс поломать перебором

Для RSA ключа минимальная безопасная длина - **2048 бит**. Всё, что короче (1024, 512) - это уже дыра 

---------

тест провожу на приложении 👉 [[0_MeetWay]]

понадобится утилита apksigner

установка тут [[apksigner]]

запуск команды для полной проверки проверки
```q
evgeniy@Evgeniys-MacBook-Pro-2 ~ % java -jar ~/android-11/lib/apksigner.jar verify --print-certs --verbose ~/Desktop/meetway.apk


Verifies
Verified using v1 scheme (JAR signing): false
Verified using v2 scheme (APK Signature Scheme v2): true
Verified using v3 scheme (APK Signature Scheme v3): false
Verified using v4 scheme (APK Signature Scheme v4): false
Verified for SourceStamp: false
Number of signers: 1
Signer #1 certificate DN: C=US, O=Android, CN=Android Debug
Signer #1 certificate SHA-256 digest: d5fd0eecf21f261559e6d1fd635fc5e762af7640c6a1655a50231e7fd3b2b2ef
Signer #1 certificate SHA-1 digest: 802e5ab8b2916eeaac119af80619c015dce47742
Signer #1 certificate MD5 digest: f4c3cd7ff5d13e84819629261249db7a

Signer #1 key algorithm: RSA 👈  ✅ - ЭТО ТО ЧТО НУЖНО!!! СТАНДАРТ
Signer #1 key size (bits): 2048  👈  ✅ - ЭТО ТО ЧТО НУЖНО!!!  СТАНДАРТ

Signer #1 public key SHA-256 digest: d2a998c20d12bba01554c111506ea3161ca70e8b3c3237d6a77ec996f25d7390
Signer #1 public key SHA-1 digest: d6b36b732097e2f7168130902fc8b4ea3837ec73
Signer #1 public key MD5 digest: 37ded2f2819126faabfa7f0ffe1c343a
evgeniy@Evgeniys-MacBook-Pro-2 ~ %
```

Signer #1 key algorithm: RSA 👈  ✅ - ЭТО ТО ЧТО НУЖНО!!! СТАНДАРТ
Signer #1 key size (bits): 2048  👈  ✅ - ЭТО ТО ЧТО НУЖНО!!!  СТАНДАРТ

-----


## вывод

Тест **ПРОЙДЕН**. Ключ подписи имеет достаточную длину (2048 бит) и не создает рисков компрометации подписи

