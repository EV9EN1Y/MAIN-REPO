
# MASTG-TEST-0214: жестко заданные криптографические ключи в файлах

в отличии от тест 0213 - где проверялся только код
здесь - нужно искать и внутри файлов!

-------

тестирую также приложение [[0_MeetWay]]

```q
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la
total 125928
-rwxr-xr-x   1 evgeniy  staff     35024 30 апр 19:00 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96 28 апр 14:29 _CodeSignature
drwxr-xr-x  16 evgeniy  staff       512 30 апр 19:00 .
drwxr-xr-x@  5 evgeniy  staff       160 30 апр 19:02 ..
-rw-r--r--   1 evgeniy  staff     14412 30 апр 18:59 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132 30 апр 18:59 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696 30 апр 18:59 Assets.car
-rw-r--r--   1 evgeniy  staff     15136 28 апр 14:28 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864 28 апр 14:29 Frameworks
-rw-r--r--   1 evgeniy  staff       996 28 апр 14:28 GoogleService-Info.plist
-rw-r--r--   1 evgeniy  staff      3901 29 апр 14:20 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664 30 апр 19:00 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41606208 30 апр 19:00 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8 28 апр 14:28 PkgInfo
drwxr-xr-x   4 evgeniy  staff       128 28 апр 14:28 TrustKit_TrustKit.bundle
-rw-r--r--   1 evgeniy  staff     49852 28 апр 14:28 words.txt
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % r2 -A ./MeetWay.debug.dylib
```


так как в прошлом тесте я сканировал файл MeetWay.debug.dylib

но сейчас тоже самое нужно сделать, но без файла MeetWay.debug.dylib

--------

и тут уже не обязательно радар использовать, все таки это не бинарник

```q


/поиск base64 строк во всех файлах (кроме больших бинарников)
grep -r -E "[A-Za-z0-9+/]{32,}=" . --exclude="*.dylib" --exclude="MeetWay" --exclude="*.car" 2>/dev/null

 /поиск ключевых слов
grep -r -i "key\|secret\|private\|api_key\|token\|password" . --include="*.plist" --include="*.json" --include="*.strings" --include="*.txt" 2>/dev/null

//поиск PEM маркеров
grep -r "BEGIN PRIVATE KEY\|BEGIN CERTIFICATE" . 2>/dev/null
```

и тотже скрипт
```js
/./.lля всех .plist и .json файлов
for file in $(find . -type f \( -name "*.plist" -o -name "*.json" -o -name "*.strings" \) 2>/dev/null); do
    echo "=== $file ==="
    strings "$file" 2>/dev/null | python3 -c "
import sys, math
for line in sys.stdin:
    s = line.strip()
    if len(s) > 20:
        freq = {}
        for c in s:
            freq[c] = freq.get(c, 0) + 1
        entropy = -sum((freq[c]/len(s))*math.log2(freq[c]/len(s)) for c in freq)
        if entropy > 4.5:
            print(f'  [ENT={entropy:.2f}] LEN={len(s)}: {s[:80]}')
    "
    echo ""
done
```

--------

результаты
```q
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % grep -r -E "[A-Za-z0-9+/]{32,}=" . --exclude="*.dylib" --exclude="MeetWay" --exclude="*.car" 2>/dev/null
./_CodeSignature/CodeResources:			jLLnWEHVMFdai9UMKSO1Npb2EtWBDr6vk5lp/7Cy/mw= эт - хеши файлов для проверки подписи кода ✅
./_CodeSignature/CodeResources:			6V7VZqpE9VjP65Q3/Fb67K2axjUKgSyVvp59Lb3KMoo=
./_CodeSignature/CodeResources:			pSEXB2ryl/8PPHny9vY2Vk8q6+dxVtdx3MWQ+jCRckw=
./_CodeSignature/CodeResources:			DeX+3VaZkl/kiTZPgirEXqHEgqEDvauEsKJH1/5aNQE=
./_CodeSignature/CodeResources:			iS3KiM9A7ViIxhNYilKEqRngEVQtaEyRZAU2FSuMvsQ=
./_CodeSignature/CodeResources:			/xtdxoyDxQSImZ02Cewe2Sfbutih0+5AqRf6TXwyctE=
./_CodeSignature/CodeResources:			эт - хеши файлов для проверки подписи кода ✅6bbG9zH4WhqVTKRP5UgUXdkIaHAxzRU19wcOU7lPRHA=
./_CodeSignature/CodeResources:			RJtVkRHIJMatcJ59+H4v1PlVugrXLy7iF/XVzUFTqaU=
./_CodeSignature/CodeResources:			hkT70adikQe80PY0CuxAz7xMd0PzbPLgGv7SkX5HwmE=
./_CodeSignature/CodeResources:			vLKwAXRoabkdijeS549V8gZkgGiMxpmxkZ1U+BRvbUI=
./_CodeSignature/CodeResources:			эт - хеши файлов для проверки подписи кода ✅5je3v4IFUUNu8Emec/91g4ibohsKxE9ljUMmaafCTBc=
./_CodeSignature/CodeResources:			AW4AVN4/YDCyemiDJeP9Tbrba/u2pu0K/WdOI9s10Go=
oyIB29YmcqtKWgYvnj3pGb1IK9KPlH1gAmA2RFXxLWw=
./_CodeSignature/CodeResources:			k9513GlJjP4I16GXPs2GFHPvdg2wUtRLZAZEfuK/WjM=эт - хеши файлов для проверки подписи кода ✅
./_CodeSignature/CodeResources:		

./_CodeSignature/CodeResources:			DahyilGqbR3JEkehfmnmMa01ChcBVRNLKqlQpR3bb94= эт - хеши файлов для проверки подписи кода ✅
./_CodeSignature/CodeResources:			o3SuyeZzlD61gQwZaIOMy5YvUtrv7Xfn4gCUMrOpHWw=
./_CodeSignature/CodeResources:			yvTIxdMK31qlE/641POlEkvnvNgG+dBhDN9VbVdgI3g=

./_CodeSignature/CodeResources:			ErfWvNX8xk1lcw06VAjmYe2f5qjiGRgSBxu4knM4jWQ=
./Frameworks/grpcpp.framework/_CodeSignature/CodeResources:			/ZAK2s+rCWhA0eeAntv4NlVXDqwpTT8AjDWRzkhFj1Y=
./Frameworks/grpcpp.framework/_CodeSignature/CodeResources:			lhQzRMSuEJWIElssQdXa9lSl+vxuI/rDf3uj0p2n53Y=
./Frameworks/grpcpp.framework/_CodeSignature/CodeResources:			PjJ9e43oKQRzhM8KBKmN7kJtghDko+MG0zg2JJE3vr8=
./Frameworks/grpcpp.framework/_CodeSignature/CodeResources:			A7LHCDOjMaKx79Ef8WjtAqjq39Xn0fvzDuzHUJpK6kc=
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:HMUfpIBvFSDJ3gyICh3WZlXi/EjJKSZp4A==
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:TBj0/VLZjmmx6BEP3ojY+x1J96relc8geMJgEtslQIxq/H5COEBkEveegeGTLg==
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:RSflMMFe8toTyyVCUZVHA4xsIcx0Qu1T/zOLjw9XARYvz6buyXAiFL39vmwLAw==
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:WQPJIrSPnNVeKtelttQKbfi3QBFGmh95DmK/D5fs4C8fF5Q=
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:+o0bJW1sj6W3YQGx0qMmoRBxna3iw/nDmVG3KwcIzi7mULKn+gpFL6Lw8g==

✅ то че внутри фреймворкоа - это норма - при условии- когда они безопасны и обновлены!!!!!

Binary file ./embedded.mobileprovision matches
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % grep -r -i "key\|secret\|private\|api_key\|token\|password" . --include="*.plist" --include="*.json" --include="*.strings" --include="*.txt" 2>/dev/null
Binary file ./GoogleService-Info.plist matches
./words.txt:donkey punch
./words.txt:honkey
./words.txt:pikey
./words.txt:Ass Monkey
./words.txt:skankey
./words.txt:honkey
./words.txt:cold turkey

// это не прикол выше - это совпадения в файле  - блэк листе плохих слов

evgeniy@Evgeniys-MacBook-Pro MeetWay.app % grep -r "BEGIN PRIVATE KEY\|BEGIN CERTIFICATE" . 2>/dev/null
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:-----BEGIN CERTIFICATE-----
./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/roots.pem:----

// это паблик сертификаты - все ок

Binary file ./Frameworks/FirebaseFirestoreInternal.framework/FirebaseFirestoreInternal matches
Binary file ./MeetWay.debug.dylib matches
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```


результат скрипта

```
=== ./TrustKit_TrustKit.bundle/Info.plist ===
  [ENT=4.53] LEN=60: DTPlatformVersionZDTSDKBuildYDTSDKNameWDTXcode\DTXcodeBuild_
  [ENT=4.62] LEN=83: "com.apple.compilers.llvm.clang.1_0U22F76XiphoneosT18.5\iphoneos18.5T1640T16F6T1

=== ./GoogleService-Info.plist ===
  [ENT=4.71] LEN=75: FirebaseAppDelegateProxyEnabled]GCM_SENDER_ID]GOOGLE_APP_ID^IS_ADS_ENABLE
  
  
  ⚠️⚠️⚠️⚠️❌❌🔴🔴🔴
  [ENT=4.79] LEN=55: NSUserNotificationAlertStyleZPROJECT_ID^STORAGE_BUCKET_
  [ENT=5.28] LEN=54: 'AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEMY-2e.AAIVARen_  👈⚠️⚠️⚠️⚠️❌❌🔴🔴🔴




=== ./Frameworks/grpcpp.framework/gRPCCertificates-Cpp.bundle/Info.plist ===
  [ENT=4.53] LEN=60: DTPlatformVersionZDTSDKBuildYDTSDKNameWDTXcode\DTXcodeBuild_
  [ENT=4.54] LEN=37: gRPCCertificates-CppTBNDLV1.69.0T????
  [ENT=4.61] LEN=83: "com.apple.compilers.llvm.clang.1_0U22F76XiphoneosT18.5\iphoneos18.5T1640T16F6T1


=== ./Frameworks/FBLPromises.framework/Info.plist ===
  [ENT=4.53] LEN=60: DTPlatformVersionZDTSDKBuildYDTSDKNameWDTXcode\DTXcodeBuild_
  [ENT=4.69] LEN=51: UIRequiredDeviceCapabilitiesV24G624Ren[FBLPromises_
  [ENT=4.70] LEN=82: "com.apple.compilers.llvm.clang.1_0U22F76XiphoneosT18.5\iphoneos18.5T1640T16F6S9


=== ./Info.plist ===
  [ENT=5.64] LEN=54: !"#$%&'()*+,-.2345=AB4CDEGNOPQRPSTUVW[\]hijklmnopq[w|~
  [ENT=4.53] LEN=60: DTPlatformVersionZDTSDKBuildYDTSDKNameWDTXcode\DTXcodeBuild_
  [ENT=4.53] LEN=48: UIBackgroundModes^UIDeviceFamily^UILaunchScreen_
  [ENT=4.59] LEN=55: $AIVARO22-2025-1.0.notification-checkV24G624RenWMeetWay
  [ENT=4.64] LEN=78: "com.apple.compilers.llvm.clang.1_0U22F76XiphoneosT18.5\iphoneos18.5T1640T16F6
  [ENT=4.54] LEN=44: +NSTemporaryExceptionAllowsInsecureHTTPLoads
```

--------------------




### вывод


  
  ⚠️⚠️⚠️⚠️❌❌🔴🔴🔴
 ` [ENT=4.79] LEN=55: NSUserNotificationAlertStyleZPROJECT_ID^STORAGE_BUCKET_
`  [ENT=5.28] LEN=54: 'AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEMY-2e.AAIVAR_en``  👈⚠️⚠️⚠️⚠️❌❌🔴🔴🔴

это хардкор вшитый в плист апи кей от Firebase!!!

то - кто его узнает - получает доступ к Firebase серверу моему!
Ключ доступен любому, кто распакует IPA

```c
 Как должно быть (варианты)

нужно удалить ключ из бандла приложения

Сервер -все на сервере! - Клиент не знает Firebase ключ. Всё идёт через мой сервер

Firebase App Check -- -Ограничить ключ только для нашего приложения

API restrictions - -- Ограничить ключ только по доменам/IP

удалить из репозитория - - -- Ключ должен быть в CI/CD, а не в кодe
```

##### `GoogleService-Info.plist` по умолчанию **не предназначен для секретов**. Это просто конфиг

-------

#### тест провален ❌


