## MASTG-TEST-0220: Использование устаревшего формата кодовой подписи

>  Цель данного теста проверить, в каком формате подписано приложение - в  **старом (v1, SHA-1)** или в **новом (v2,  SHA-256)**

> Если приложение подписано старым форматом (v1), злоумышленнику будет проще переподписать и модифицировать приложение (например, встроить троян или инструмент анализа), и пользователь не заметит подмены

Новый формат подписи (v2) использует **более стойкую криптографию (SHA-256)** и содержит дополнительные механизмы защиты от подделки, что сильно усложняет внедрение чужого кода (Frida) без нарушения подписи

то есть хорошая криптография помогает сопротивляться тестам, которые я проводил:
[[_R_проверка-целостности-файлов-и-кода(патч-ipa)]]


Современные версии Xcode (  примерно от ios 15 ) по умолчанию используют актуальный формат подписи (SHA-256). Поэтому специально настраивать ничего не нужно, достаточно просто проверить

---------

#### как лечить это?

- использовать новейшие версии xcode (т.к он сам подписвывает)
- использовать современный таргет ios 14+
- не использовать ручной `codesign` с флагами (это если вручную переподписать)

начиная с ios 15  xcode сам подскажет - если подпись устаревшая


------

### как быстро проверить 

едем на сайт APPLE
https://developer.apple.com/documentation/xcode/using-the-latest-code-signature-format?language=objc

и там прямым текстом пишут
```c
Look in the output for a string such as `CodeDirectory v=20500`. For any value of `v` less than `20400`, you need to re-sign your app.

If the code directory value is 20400 or higher, and the app does not install on iOS 15, iPadOS 15, tvOS 15, or watchOS 8, confirm that the system has added DER entitlements to the app’s signature. Also confirm that you have correctly signed any nested code, for example, an app extension, a framework, or a bundled watchOS app
```

то есть минималка - это 20400

теперь иду в свой макбук:
проверяю :

команда 
```shell
codesign -dv /путь к приложению 

-------

нужно смотреть на   CodeDirectory

----------

evgeniy@Evgeniys-MacBook-Pro ~ % codesign -dv /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-*/Build/Products/Debug-iphoneos/MeetWay.app
Executable=/Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/MeetWay
Identifier=AIVARO22-2025-1.0
Format=app bundle with Mach-O thin (arm64)
⭐️ 👉 CodeDirectory v=20400 size=917 flags=0x0(none) hashes=18+7 👈✅ location=embedded
Signature size=4791
Signed Time=25 Apr 2026 at 16:07:58
Info.plist entries=44
TeamIdentifier=MF5MXG95XN
Sealed Resources version=2 rules=10 files=128
Internal requirements count=1 size=188
/Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-crywkrswaksuszcmfayjzyqvwnxq/Build/Products/Debug-iphoneos/MeetWay.app: bundle format unrecognized, invalid, or unsuitable
```

✅ вижу 20400 - все еще считается нормальным!
✅✅✅ тест пройден!



вот углубленная проверка
```shell
codesign -d --verbose=5 /путь к приложению

------------------

evgeniy@Evgeniys-MacBook-Pro MeetWay.app % codesign -d --verbose=5 /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-*/Build/Products/Debug-iphoneos/MeetWay.app
Executable=/Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-adcrpxpsonfrkahkdzxzzbselvsp/Build/Products/Debug-iphoneos/MeetWay.app/MeetWay
Identifier=AIVARO22-2025-1.0
Format=app bundle with Mach-O thin (arm64)
CodeDirectory v=20400 size=917 flags=0x0(none) hashes=18+7 location=embedded
VersionPlatform=2
VersionMin=1115648
VersionSDK=1180928
Hash type=sha256 size=32
CandidateCDHash sha256=0f18648da27b3151a2def35f7fefa5e12b74b7ba
CandidateCDHashFull sha256=0f18648da27b3151a2def35f7fefa5e12b74b7ba34ea10650db418ad839a8caf
Hash choices=sha256
CMSDigest=0f18648da27b3151a2def35f7fefa5e12b74b7ba34ea10650db418ad839a8caf
CMSDigestType=2
Executable Segment base=0
Executable Segment limit=49152
Executable Segment flags=0x11
Page size=4096
    -7=2e71925730b0c0af1a516bfa4ca8bbdcea1f69aa8eff2c3d7f004e1f87dd5d1f
    👉 🟢✅ здесь важно наличие строки -7=2e7192573.. если есть - все окей
    
    -6=0000000000000000000000000000000000000000000000000000000000000000
    -5=e29c6492ee50887686712ae190e421fae45ab296c2e0deee8702f49d916207c6
    -4=0000000000000000000000000000000000000000000000000000000000000000
    -3=37245fc917656b4476e0f8c454e527c761776e5f51c551508e73cf63ead241c9
    -2=021b2ef626164f0920e4c15787e600552856e8535ba8170951478d259b540c77
CDHash=0f18648da27b3151a2def35f7fefa5e12b74b7ba
Signature size=4791
Authority=Apple Development: Evgenyi Chernikov (CJJ8HX2K8L)
Authority=Apple Worldwide Developer Relations Certification Authority
Authority=Apple Root CA
Signed Time=25 Apr 2026 at 16:07:58
Info.plist entries=44
TeamIdentifier=MF5MXG95XN
Sealed Resources version=2 rules=10 files=128
Internal requirements count=1 size=188
/Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-crywkrswaksuszcmfayjzyqvwnxq/Build/Products/Debug-iphoneos/MeetWay.app: bundle format unrecognized, invalid, or unsuitable


```

эппл говрит, на этомже сайте:
```c
To check whether the app has the DER entitlements, look for the hash list under Page size in the signature. If `-5`contains a value and `-7` contains a zero value, or is not present, you need to re-sign your app to include the new DER entitlements.
Valid signature:
Page size=4096
   -7=f4c7c0ae394247097dca9b19333001200747691e1d9e25ec0cf0f35a8ade21f3

типо если нет -7 или там пусто - то нужно переподписать на современнома макбуке с современным xcode - самое простое!

```

✅вижу  строку -7=2e71925730.... значит права DER установлены!
✅✅✅ тест пройден  !!

код подписи соответствует современным требованиям, использует стойкий алгоритм SHA-256 и включает обязательные DER-прав

-----

