# MASTG-TEST-0213: Использование жестко заданных криптографических ключей в коде


 ПРОВЕРЯЕТ --- не вшиты ли ключи шифрования прямо в исходный код/бинарник

соответственно, если ключ в коде - любой, кто скачает IPA, достанет ключ за 5 минут через `strings` или radare2. Шифрование становится бесполезным


вот так могут быть вшиты в код ключи
```
// строка
let key = "MySecretKey123456"

// массив байт
let keyBytes: [UInt8] = [0x2b, 0x7e, 0x15, 0x16, ...]

// Base 64
let keyBase64 = "dGhpc2lzYXRlc3RrZXk="

// нex строка
let keyHex = "2b7e151628aed2a6abf7158809cf4f3c"
```

-------

тест буду проводить на ios приложении [[0_MeetWay]]
(это рабочая соц сеть)


запускаю радар2 на бинарнике

```q
evgeniy@Evgeniys-MacBook-Pro ~ % cd /Users/evgeniy/Desktop/проверка\ face\ \+\ блокир\ экрана/MeetWay.app
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


выполняю поиск устаревших алгоритмов


тестировать нужно так:
```q


#  потенциальные ключи по именам переменных
izz | grep -iE "key|secret|pass|token|api_key|private|aes|gcm|chaCha|Encrypt|Decrypt"
(❌ но это жсть - все подряд)


#  длинные hex-строки (32+ символов) — типичный ключ
izz | grep -E "[0-9a-f]{32,64}" -i
( ❌ но это жсть - тоже  все подряд)

тоже самое - но с отсеиванием по эентропии

#  base64 строки (длинные)
izz | grep -E "[A-Za-z0-9+/]{32,}=" -i
( ✅ НОРМАЛЬНО НАХОДИТ)
--------

#  массивы UInt8
/ [UInt8]

#  Data инициализацию
/ Data\(


------------------

если есть находки - то проверить контекст использования 
is | grep CCCrypt
axt sym.imp.CCCrypt
pd 20
```

Скрипт, который ищет строки с высокой «случайностью» (энтропией)
```python
strings MeetWay.debug.dylib | python3 -c "
import sys, math
for line in sys.stdin:
    s = line.strip()
    if len(s) > 20:
        freq = {}
        for c in s:
            freq[c] = freq.get(c, 0) + 1
        entropy = -sum((freq[c]/len(s))*math.log2(freq[c]/len(s)) for c in freq)
        if entropy > 4.5:
            print(f'[ENTROPY {entropy:.2f}] LEN={len(s)}: {s[:100]}')
"

(можно поиграться длинной и стпенью)

```




-------

результаты 
```q
[ENTROPY 4.62] LEN=110: SS4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2SG0H7
[ENTROPY 4.64] LEN=108: SS4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2SG0H7
[ENTROPY 4.66] LEN=113: SDyS2S4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2S
[ENTROPY 4.67] LEN=115: ySDyS2S4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2
[ENTROPY 4.61] LEN=112: SS_SS4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2SG
[ENTROPY 4.69] LEN=116: ySDyS2S4name_SS10professionSS11profession2SS8languageSS7countrySS9avatarURLSS6ratingSS8nickNameSDyS2
[ENTROPY 5.37] LEN=120: https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9
[ENTROPY 4.75] LEN=44: kyHvcBfJA5The/fNkQH0q/IVw/uk5bfphH41bQmDATo=
[ENTROPY 5.08] LEN=44: C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ=
[ENTROPY 5.00] LEN=44: +vLEyBQERqRwpgiGwEi7Dx6jujKTEdoJzr4CSmYXCz0=
[ENTROPY 4.80] LEN=40: YCMZYgs2XIJupQenfJbRsqeCNLN5xjaKtGj17ep2
[ENTROPY 5.95] LEN=62: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
[ENTROPY 5.95] LEN=62: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
[ENTROPY 4.60] LEN=38: v24@?0@"FIRQuerySnapshot"8@"NSError"16
[ENTROPY 4.67] LEN=41: v24@?0@"FIRDocumentSnapshot"8@"NSError"16
[ENTROPY 4.61] LEN=40: v16@?0@"UIGraphicsImageRendererContext"8
[ENTROPY 4.67] LEN=54: IOS16/IVAARO1 2/IVAARO/MANAGER_S/VideoCompanents.swift
[ENTROPY 4.68] LEN=39: v40@0:8@16^{opaqueCMSampleBuffer=}24@32
[ENTROPY 4.89] LEN=107: vkewjrbg@#$%$**)(_&^%4257(&^&^$%&^**_ch35ha56ht_encryp56ti6534yon_safhd457_202346hy5_4h4v3_ertb356ui
[ENTROPY 4.59] LEN=28: v24@?0q8@"NSURLCredential"16
[ENTROPY 4.94] LEN=44: Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=
[ENTROPY 4.51] LEN=42: v24@?0@"AWSS3PutObjectOutput"8@"NSError"16
[ENTROPY 4.53] LEN=42: v24@?0@"AWSS3GetObjectOutput"8@"NSError"16
[ENTROPY 4.55] LEN=45: v24@?0@"AWSS3DeleteObjectOutput"8@"NSError"16
[ENTROPY 4.66] LEN=43: v24@?0@"AWSS3HeadObjectOutput"8@"NSError"16
[ENTROPY 4.79] LEN=44: CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=
[ENTROPY 4.58] LEN=45: v24@?0@"<NSItemProviderReading>"8@"NSError"16
[ENTROPY 4.77] LEN=66: arn:aws:sns::b1ggdrk2tndoa2t1f6jf:app/APNS_SANDBOX/testSandboxPush
[ENTROPY 4.80] LEN=69: _TtC7MeetWayP33_94BB64A8071F57B6B2C38F32C62C60E219ResourceBundleClass
[ENTROPY 4.80] LEN=79: void swizzle(__unsafe_unretained Class, SEL, __strong RSSwizzleImpFactoryBlock)
[ENTROPY 4.76] LEN=63: BOOL blockIsCompatibleWithMethodType(__strong id, const char *)
[ENTROPY 4.73] LEN=184: /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-bsqivqjhwnkdendilvhdxrqbtnoy/SourcePackage
[ENTROPY 4.73] LEN=180: /Users/evgeniy/Library/Developer/Xcode/DerivedData/IVAARO-bsqivqjhwnkdendilvhdxrqbtnoy/SourcePackage
[ENTROPY 4.75] LEN=168: TSKTrustEvaluationResult verifyPublicKeyPin(SecTrustRef _Nonnull, NSString *__strong _Nonnull, NSSet
[ENTROPY 4.60] LEN=72: @"NSURLConnection"32@?0@8@"NSURLRequest"16@"<NSURLConnectionDelegate>"24
[ENTROPY 4.66] LEN=75: @"NSURLConnection"36@?0@8@"NSURLRequest"16@"<NSURLConnectionDelegate>"24B32
[ENTROPY 4.66] LEN=90: v40@?0@8@"NSURLRequest"16@"NSOperationQueue"24@?<v@?@"NSURLResponse"@"NSData"@"NSError">32
[ENTROPY 4.61] LEN=76: @"NSURLSession"40@?0@8@"NSURLSessionConfiguration"16@24@"NSOperationQueue"32
[ENTROPY 4.70] LEN=65: v32@?0@"TSKPinningValidatorResult"8@"NSString"16@"NSDictionary"24
[ENTROPY 4.51] LEN=180: TrustKit was initialized with less than %lu pins (ie. no backup pins) for domain %@. This might bric
[ENTROPY 4.61] LEN=37: $s7MeetWay15LoadingOverlay2V4bodyQrvp
[ENTROPY 4.83] LEN=61: $s7SwiftUI4ViewP7MeetWayE19addSwipeBackGesture6actionQryyc_tF
[ENTROPY 5.00] LEN=90: $s7MeetWay12RegistrationV12AuthResponseV10CodingKeys33_13EB04D5D94B59450665E9733D40AF31LLO
[ENTROPY 4.97] LEN=85: $s7MeetWay12RegistrationV8UserDataV10CodingKeys33_13EB04D5D94B59450665E9733D40AF31LLO
[ENTROPY 4.93] LEN=78: $s7MeetWay12RegistrationV10headerView33_13EB04D5D94B59450665E9733D40AF31LLQrvp
[ENTROPY 4.96] LEN=76: $s7MeetWay12RegistrationV9nameField33_13EB04D5D94B59450665E9733D40AF31LLQrvp
[ENTROPY 5.04] LEN=81: $s7MeetWay12RegistrationV13nickNameField33_13EB04D5D94B59450665E9733D40AF31LLQrvp
[ENTROPY 4.86] LEN=82: $s7MeetWay12RegistrationV14registerButton33_13EB04D5D94B59450665E9733D40AF31LLQrvp
[ENTROPY 5.12] LEN=82: $s7MeetWay12RegistrationV14policyCheckbox33_13EB04D5D94B59450665E9733D40AF31LLQrvp
[ENTROPY 4.88] LEN=98: $s7MeetWay9ViewModelC7fbtnutn6focuse4size5iconS7idWhose6actionQrSb_12CoreGraphics7CGFloatVS2SyyctF
[ENTROPY 5.04] LEN=126: $s7MeetWay9ViewModelC6tapBar9sizeFrame0G05iconS11collorFront0J4Back6actionQr12CoreGraphics7CGFloatV_
[ENTROPY 5.02] LEN=118: $s7MeetWay17VideoRecorderViewV08recordedc7PreviewE033_7A65C1B32CDCCBA94908ADB7D8ED25CBLL8videoURLQr1
[ENTROPY 4.53] LEN=40: $s7MeetWay18CustomProgressViewV4bodyQrvp
[ENTROPY 4.52] LEN=41: $s7MeetWay19FullscreenVideoViewV4bodyQrvp
[ENTROPY 4.60] LEN=46: $s7MeetWay24AllCountryScreenUserViewV4bodyQrvp
[ENTROPY 4.83] LEN=70: $s7MeetWay8GoalCellV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
[ENTROPY 4.80] LEN=74: $s7MeetWay11LocatesUserV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
[ENTROPY 4.85] LEN=75: $s7MeetWay12LocationInfoV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
[ENTROPY 4.86] LEN=67: $s7MeetWay5placeV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
[ENTROPY 4.91] LEN=82: $s7MeetWay19postAchivementsDataV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
[ENTROPY 4.75] LEN=45: $s7MeetWay23DynamicHeightTextEditorV4bodyQrvp
[ENTROPY 4.69] LEN=41: $s7MeetWay19FullscreenImageViewV4bodyQrvp
[ENTROPY 4.67] LEN=45: $s7MeetWay23GoalAchievementPostViewV4bodyQrvp
[ENTROPY 4.54] LEN=36: $s7MeetWay14LisAfterSearchV4bodyQrvp
[ENTROPY 4.57] LEN=30: $s7MeetWay9GoalScrinV4bodyQrvp
[ENTROPY 4.54] LEN=37: $s7MeetWay15AudioPlayerViewV4bodyQrvp
[ENTROPY 4.54] LEN=32: $s7MeetWay10FirstScrinV4bodyQrvp
[ENTROPY 4.64] LEN=44: $s7MeetWay22CustomAlertViewOnboardV4bodyQrvp
[ENTROPY 4.51] LEN=34: $s7MeetWay12EnterAccountV4bodyQrvp
[ENTROPY 4.91] LEN=84: $s7MeetWay15AuthButtonStyleV8makeBody13configurationQr7SwiftUI0dE13ConfigurationV_tF
[ENTROPY 4.58] LEN=38: $s7MeetWay16AllCountryScreenV4bodyQrvp
[ENTROPY 4.53] LEN=44: $s7MeetWay22RecordingIndicatorViewV4bodyQrvp
[ENTROPY 4.51] LEN=41: $s7MeetWay19AudioVisualizerViewV4bodyQrvp
[ENTROPY 5.01] LEN=110: $s7MeetWay5ChatsV15MessageItemViewV011mainContentF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentU
[ENTROPY 5.15] LEN=112: $s7MeetWay5ChatsV15MessageItemViewV012videoAndTextdF033_58B9898145204CA89E0CDD64611B323BLL13isCurren
[ENTROPY 5.16] LEN=115: $s7MeetWay5ChatsV15MessageItemViewV05videodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser8sh
[ENTROPY 5.01] LEN=93: $s7MeetWay5ChatsV15MessageItemViewV012videoContentF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.04] LEN=93: $s7MeetWay5ChatsV15MessageItemViewV012compactVideoF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.02] LEN=94: $s7MeetWay5ChatsV15MessageItemViewV013expandedVideoF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.20] LEN=113: $s7MeetWay5ChatsV15MessageItemViewV014videoThumbnailF033_58B9898145204CA89E0CDD64611B323BLL5imageQrS
[ENTROPY 5.19] LEN=110: $s7MeetWay5ChatsV15MessageItemViewV016videoPlaceholderF033_58B9898145204CA89E0CDD64611B323BLL9isLoad
[ENTROPY 5.01] LEN=91: $s7MeetWay5ChatsV15MessageItemViewV010videoErrorF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.09] LEN=104: $s7MeetWay5ChatsV15MessageItemViewV05audiodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrS
[ENTROPY 5.16] LEN=115: $s7MeetWay5ChatsV15MessageItemViewV05photodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser8sh
[ENTROPY 5.20] LEN=124: $s7MeetWay5ChatsV15MessageItemViewV013photoAndAudiodF033_58B9898145204CA89E0CDD64611B323BLL13isCurre
[ENTROPY 5.08] LEN=107: $s7MeetWay5ChatsV15MessageItemViewV08allThreedF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser
[ENTROPY 5.11] LEN=115: $s7MeetWay5ChatsV15MessageItemViewV016reactionsOverlayF033_58B9898145204CA89E0CDD64611B323BLL13isCur
[ENTROPY 5.15] LEN=112: $s7MeetWay5ChatsV15MessageItemViewV012audioAndTextdF033_58B9898145204CA89E0CDD64611B323BLL13isCurren
[ENTROPY 5.04] LEN=103: $s7MeetWay5ChatsV15MessageItemViewV04textdF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb
[ENTROPY 4.99] LEN=93: $s7MeetWay5ChatsV15MessageItemViewV012photoContentF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.06] LEN=93: $s7MeetWay5ChatsV15MessageItemViewV012audioLoadingF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.02] LEN=91: $s7MeetWay5ChatsV15MessageItemViewV010audioErrorF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 5.11] LEN=97: $s7MeetWay5ChatsV15MessageItemViewV016audioPlaceholderF033_58B9898145204CA89E0CDD64611B323BLLQryF
[ENTROPY 4.96] LEN=112: $s7MeetWay5ChatsV15MessageItemViewV013messageStatusF033_58B9898145204CA89E0CDD64611B323BLL13isCurren
[ENTROPY 4.69] LEN=47: $s7MeetWay5ChatsV18ReactionPickerViewV4bodyQrvp
[ENTROPY 5.15] LEN=98: $s7MeetWay5ChatsV18ReactionPickerViewV11emojiButton33_58B9898145204CA89E0CDD64611B323BLL0G0QrSS_tF
[ENTROPY 4.82] LEN=93: $s7SwiftUI4ViewP7MeetWayE12cornerRadius_7cornersQr12CoreGraphics7CGFloatV_So12UIRectCornerVtF
[ENTROPY 4.57] LEN=40: $s7MeetWay18GoalScreenUserViewV4bodyQrvp
[ENTROPY 4.78] LEN=69: $s7MeetWay19ResourceBundleClass33_94BB64A8071F57B6B2C38F32C62C60E2LLC
[ENTROPY 4.62] LEN=47: T@"NSObject<OS_dispatch_queue>",&,N,V_lockQueue
[ENTROPY 4.82] LEN=67: T@"NSObject<OS_dispatch_queue>",&,N,V_pinningValidatorCallbackQueue
[ENTROPY 4.75] LEN=61: T@"NSObject<OS_dispatch_queue>",R,N,V_validationCallbackQueue
[ENTROPY 4.66] LEN=39: T@"NSMutableDictionary",&,N,V_spkiCache
[ENTROPY 4.68] LEN=61: T@"NSObject<OS_dispatch_queue>",&,N,V_pinFailureReporterQueue
[ENTROPY 4.61] LEN=55: T@"NSObject<OS_dispatch_queue>",&,N,V_reportsCacheQueue
[ENTROPY 4.51] LEN=44: v56@0:8@16{CGRect={CGPoint=dd}{CGSize=dd}}24
[ENTROPY 4.56] LEN=42: B32@0:8@"UIApplication"16@"NSDictionary"24
[ENTROPY 4.54] LEN=35: B32@0:8@"UIApplication"16@"NSURL"24
[ENTROPY 4.55] LEN=51: B48@0:8@"UIApplication"16@"NSURL"24@"NSString"32@40
[ENTROPY 4.63] LEN=52: B40@0:8@"UIApplication"16@"NSURL"24@"NSDictionary"32
[ENTROPY 4.50] LEN=31: v40@0:8@"UIApplication"16q24d32
[ENTROPY 4.86] LEN=59: v56@0:8@"UIApplication"16{CGRect={CGPoint=dd}{CGSize=dd}}24
[ENTROPY 4.59] LEN=56: v32@0:8@"UIApplication"16@"UIUserNotificationSettings"24
[ENTROPY 4.52] LEN=37: v32@0:8@"UIApplication"16@"NSError"24
[ENTROPY 4.56] LEN=42: v32@0:8@"UIApplication"16@"NSDictionary"24
[ENTROPY 4.69] LEN=71: v48@0:8@"UIApplication"16@"NSString"24@"UILocalNotification"32@?<v@?>40
[ENTROPY 4.69] LEN=81: v56@0:8@"UIApplication"16@"NSString"24@"NSDictionary"32@"NSDictionary"40@?<v@?>48
[ENTROPY 4.69] LEN=64: v48@0:8@"UIApplication"16@"NSString"24@"NSDictionary"32@?<v@?>40
[ENTROPY 4.74] LEN=88: v56@0:8@"UIApplication"16@"NSString"24@"UILocalNotification"32@"NSDictionary"40@?<v@?>48
[ENTROPY 4.75] LEN=52: v40@0:8@"UIApplication"16@"NSDictionary"24@?<v@?Q>32
[ENTROPY 4.56] LEN=35: v32@0:8@"UIApplication"16@?<v@?Q>24
[ENTROPY 4.77] LEN=65: v40@0:8@"UIApplication"16@"UIApplicationShortcutItem"24@?<v@?B>32
[ENTROPY 4.65] LEN=47: v40@0:8@"UIApplication"16@"NSString"24@?<v@?>32
[ENTROPY 4.65] LEN=66: v40@0:8@"UIApplication"16@"NSDictionary"24@?<v@?@"NSDictionary">32
[ENTROPY 4.52] LEN=66: v40@0:8@"UIApplication"16@"INIntent"24@?<v@?@"INIntentResponse">32
[ENTROPY 4.52] LEN=38: B32@0:8@"UIApplication"16@"NSString"24
[ENTROPY 4.64] LEN=67: @"UIViewController"40@0:8@"UIApplication"16@"NSArray"24@"NSCoder"32
[ENTROPY 4.65] LEN=37: B32@0:8@"UIApplication"16@"NSCoder"24
[ENTROPY 4.65] LEN=37: v32@0:8@"UIApplication"16@"NSCoder"24
[ENTROPY 4.75] LEN=63: B40@0:8@"UIApplication"16@"NSUserActivity"24@?<v@?@"NSArray">32
[ENTROPY 4.50] LEN=50: v40@0:8@"UIApplication"16@"NSString"24@"NSError"32
[ENTROPY 4.61] LEN=44: v32@0:8@"UIApplication"16@"NSUserActivity"24
[ENTROPY 4.67] LEN=45: v32@0:8@"UIApplication"16@"CKShareMetadata"24
[ENTROPY 4.56] LEN=65: v40@0:8@"UNUserNotificationCenter"16@"UNNotification"24@?<v@?Q>32
[ENTROPY 4.59] LEN=72: v40@0:8@"UNUserNotificationCenter"16@"UNNotificationResponse"24@?<v@?>32
[ENTROPY 4.68] LEN=39: v40@0:8@16^{opaqueCMSampleBuffer=}24@32
[ENTROPY 4.85] LEN=77: v40@0:8@"AVCaptureOutput"16^{opaqueCMSampleBuffer=}24@"AVCaptureConnection"32
[ENTROPY 4.65] LEN=65: v48@0:8@"AVCaptureFileOutput"16@"NSURL"24@"NSArray"32@"NSError"40
[ENTROPY 4.68] LEN=53: v40@0:8@"AVCaptureFileOutput"16@"NSURL"24@"NSArray"32
[ENTROPY 4.98] LEN=63: v64@0:8@"AVCaptureFileOutput"16@"NSURL"24{?=qiIq}32@"NSArray"56
[ENTROPY 4.78] LEN=50: B48@0:8@"UITextView"16{_NSRange=QQ}24@"NSString"40
[ENTROPY 4.85] LEN=57: @"UIMenu"48@0:8@"UITextView"16{_NSRange=QQ}24@"NSArray"40
[ENTROPY 4.76] LEN=59: v32@0:8@"UITextView"16@"<UIEditMenuInteractionAnimating>"24
[ENTROPY 4.63] LEN=77: v40@0:8@"UITextView"16@"UITextItem"24@"<UIContextMenuInteractionAnimating>"32
[ENTROPY 4.84] LEN=46: @"NSArray"40@0:8@"UITextView"16{_NSRange=QQ}24
[ENTROPY 4.93] LEN=50: B56@0:8@"UITextView"16@"NSURL"24{_NSRange=QQ}32q48
[ENTROPY 4.98] LEN=61: B56@0:8@"UITextView"16@"NSTextAttachment"24{_NSRange=QQ}32q48
[ENTROPY 4.84] LEN=47: B48@0:8@"UITextView"16@"NSURL"24{_NSRange=QQ}32
[ENTROPY 4.89] LEN=58: B48@0:8@"UITextView"16@"NSTextAttachment"24{_NSRange=QQ}32
[ENTROPY 4.64] LEN=57: v32@0:8@"UITextView"16@"UITextFormattingViewController"24
[ENTROPY 4.57] LEN=44: v32@0:8@"UITextView"16@"UIInputSuggestion"24
[ENTROPY 4.86] LEN=54: v48@0:8@"UIScrollView"16{CGPoint=dd}24N^{CGPoint=dd}40
[ENTROPY 4.90] LEN=85: v40@0:8@"NSURLSession"16@"NSURLAuthenticationChallenge"24@?<v@?q@"NSURLCredential">32
[ENTROPY 4.52] LEN=28: v32@0:8@"AVAudioPlayer"16Q24
[ENTROPY 4.57] LEN=87: v48@0:8@"NSURLSession"16@"NSURLSessionTask"24@"NSURLRequest"32@?<v@?q@"NSURLRequest">40
[ENTROPY 4.65] LEN=108: v56@0:8@"NSURLSession"16@"NSURLSessionTask"24@"NSHTTPURLResponse"32@"NSURLRequest"40@?<v@?@"NSURLReq
[ENTROPY 4.90] LEN=106: v48@0:8@"NSURLSession"16@"NSURLSessionTask"24@"NSURLAuthenticationChallenge"32@?<v@?q@"NSURLCredenti
[ENTROPY 4.75] LEN=70: v40@0:8@"NSURLSession"16@"NSURLSessionTask"24@?<v@?@"NSInputStream">32
[ENTROPY 4.80] LEN=73: v48@0:8@"NSURLSession"16@"NSURLSessionTask"24q32@?<v@?@"NSInputStream">40
[ENTROPY 4.53] LEN=54: v56@0:8@"NSURLSession"16@"NSURLSessionTask"24q32q40q48
[ENTROPY 4.64] LEN=60: v32@0:8@"NSURLConnection"16@"NSURLAuthenticationChallenge"24
[ENTROPY 4.55] LEN=52: B32@0:8@"NSURLConnection"16@"NSURLProtectionSpace"24
[ENTROPY 4.51] LEN=35: @56@0:8@16^{__SecTrust=}24q32q40d48
```


также базовая проверка через радар
```q
[0x00004000]> izz | grep -E "[A-Za-z0-9+/]{32,}=" -i
46022  0x00a08d10 0x00a08d10 44   45   6.__TEXT.__cstring         ascii   kyHvcBfJA5The/fNkQH0q/IVw/uk5bfphH41bQmDATo=
46023  0x00a08d40 0x00a08d40 44   45   6.__TEXT.__cstring         ascii   C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ=
46025  0x00a08d90 0x00a08d90 44   45   6.__TEXT.__cstring         ascii   +vLEyBQERqRwpgiGwEi7Dx6jujKTEdoJzr4CSmYXCz0=
47212  0x00a15b70 0x00a15b70 44   45   6.__TEXT.__cstring         ascii   Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=
47597  0x00a1a050 0x00a1a050 44   45   6.__TEXT.__cstring         ascii   CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=
[0x00004000]>


```

нужно проверить  - так как есть подохрения
```
vkewjrbg@#$%$**)(_&^%4257(&^&^$%&^**_ch35cha56ht_encryp56ti6534yon_safhd457_202346hy5_4h4v3_ertb356ui
❌ критично - вшитая соль (проверил через xcode - используется для шифр чатов)

----------
[ENT=5.37] LEN=120: https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg
✅норм  - это базовый лого - но палится адрес - не критично и не по теме

----------------
[ENT=4.75] LEN=44: kyHvcBfJA5The/fNkQH0q/IVw/uk5bfphH41bQmDATo=
[ENT=5.08] LEN=44: C5+lpZ7ncVjM8Y1E6b+dFTT2hyhCgwOxRlN96VCiJmQ=
[ENT=5.00] LEN=44: +vLEyBQERqRwpgiGwEi7Dx6jujKTEdoJzr4CSmYXCz0=
[ENT=4.94] LEN=44: Y5SLODfgYI/t+Z5+fDLg9hTWx4FaowrGml283DdTJGw=
[ENT=4.79] LEN=44: CjCkCzlbdnbdcag1zssSF7pZ4FJhzyp1xWO2WOuIwZg=
✅норм - это публичные ключи для SSL pinning (библиотека TrustKit)


[ENT=4.80] LEN=40: YCMZYgs2XIJupQenfJbRsqeCNLN5xjaKtGj17ep2
❌ не по теме = но не норм - это яндекс ключ!
------------


[ENT=4.80] LEN=69: _TtC7MeetWayP33_94BB64A8071F57B6B2C38F32C62C60E219ResourceBundleClass
✅ вообще - какой -то внутренний файл

```

-----
 тест провален 

#### ❌ критично - вшитая соль

```q
vkewjrbg@#$%$**)(_&^%4257(&^&^$%&^**_ch35cha56ht_encryp56ti6534yon_safhd457_202346hy5_4h4v3_ertb356ui
❌ критично - вшитая соль (проверил через xcode - используется для шифр чатов)
```

-------

### вывод

1. **Соль для шифрования чатов** (критично)
   - Значение: `vkewjrbg@#$%$**)(_&^%4257...`
   - Контекст: используется для шифрования сообщений в чатах
   - Риск: все пользователи используют одинаковую соль, что делает шифрование уязвимым


- Соль должна генерироваться случайно для каждого пользователя
- API ключи должны храниться на сервере или в Keychain, не в коде
- Использовать CryptoKit для безопасной генерации ключей/солей
- ё


2. **Яндекс API ключ**
   - Значение: `YCMZYgs2XIJupQenfJbRsqeCNLN5xjaKtGj17ep2`
   - Тип: hardcoded API ключ


❌❌❌❌ Тест не пройден ❌❌❌❌

