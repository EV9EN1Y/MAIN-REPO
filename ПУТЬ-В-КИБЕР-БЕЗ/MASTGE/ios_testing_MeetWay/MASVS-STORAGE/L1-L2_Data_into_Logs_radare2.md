# MASTG-TEST-0297
 Insertion of Sensitive Data into Logs


-----------

🟡 `(SAST)`
#### статический анализ бинарника приложения с помощью radare2

**Суть теста:** Проверка, что в коде нет вызовов `NSLog`, `NSAssert`, `NSCAssert`, `print` и `printf` и т.д., в которые передаются пароли, токены или ключи

---------

###  Инструкция: как выполнить проверку

 понадобится: **бинарник приложения** (файл `MeetWay` внутри `.app` или из `.ipa`) и инструмент статического анализа **radare2** (уже установлен)

#### Шаг 1. Найдите бинарник приложения

----------

👉  - Если у вас есть исходники и Xcode:
(это легко - так как мой бинарник будет не шифрованный)

вот как получить ipa при наличии исходника + xcode
```c
-Открываешь свой проект в Xcode.
- Выбираешь Product → Archive
- В открывшемся окне Organizer выбираешь свежий архив и жмешь Distribute App
Выбираешь Development или Ad Hoc
   
- Xcode сам создаст .ipa файл
  
  НО ДЛЯ ТАКОГО СПОСОБА НУЖНА ПОДПИСКА app developer
```

 Перейдите в папку со скомпилированным приложением
```c
cd ~/Library/Developer/Xcode/DerivedData/MeetWay-*/Build/Products/Debug-iphoneos/
```
Бинарник называется так же, как и приложение (например, MeetWay)

------


👉  -  Если у вас есть `.ipa` файл:
(это уже сложно, если например, просто скачать с AppStore - то чтобы распаковать чужой IPA нужен джейлбрейк, так как файл будет шифрованный, и расшифровать можно например frida-ios-dump на устройстве с джейлбрейк)

 Распакуйте .ipa
```q
unzip MeetWay.ipa -d MeetWay_extracted
```
Бинарник внутри: MeetWay_extracted/Payload/MeetWay.app/MeetWay


----------

#### Шаг 2. Откройте бинарник в radare2

 ```q
 # -A = автоматический анализ (важно!)
# -q = не открывать интерактивный режим (сразу выполним команды)

r2 -A -q -c "/ NSLog" ./MeetWay 2>/dev/null
 ```

**Что делает команда:** Ищет все вхождения строки `NSLog` в бинарнике. Вывод покажет адреса и строки кода

#### Шаг 3. Поиск других логирующих функций

```q
# Поиск NSAssert
r2 -A -q -c "/ NSAssert" ./MeetWay 2>/dev/null
# Поиск printf
r2 -A -q -c "/ printf" ./MeetWay 2>/dev/null
# Поиск os_log (современное API)
r2 -A -q -c "/ os_log" ./MeetWay 2>/dev/null
# Поиск специфичных для Swift (если приложение на Swift)
r2 -A -q -c "/ print" ./MeetWay 2>/dev/null
```

#### Шаг 4. Просмотр контекста (что именно логируется)

Для каждого найденного адреса (например, `0x100004a2c`) посмотрите окружающий код:
```q
# -A анализ уже есть, просто открываем бинарник в режиме просмотра
r2 -A ./MeetWay
[0x100004a2c]> pd 10   # показать 10 строк кода по этому адресу
[0x100004a2c]> V       # визуальный режим (стрелки, Enter для навигации)
```
**На что обратить внимание:** В строках вокруг вызова логирования ищите переменные с названиями `password`, `token`, `key`, `user`, `email`, `myID` и т.д


--------

#### Шаг 5. Альтернативный путь: поиск через строки (strings)

Быстрый способ найти все логируемые строки в бинарнике:
```q
strings ./MeetWay | grep -i "log\|debug\|info\|error" | head -50
```

Если увидите осмысленные фразы, содержащие `token`, `password` = это верный признак утечки


---------


--------

### так как первые два способа мне сейчас недоступны
так как у меня нет ни подписки apple dev  ни джейлбрейкнутого устройства

то за счет того, что я могу запустить приложение на своем устройстве через xcode - при компиляции приложения в xcode в папке **DerivedData** хранится скомпилированное приложение, его нужно найти по пути ~/Library/Developer/Xcode/DerivedData через finder например и достать от туда бинарник приложения, будет выглядеть как расширение .app

-----

1 - запуск xcode + запуск приложение на телефон через xcode
2 -нахожу в finder =  Cmd + Shift + G = 
~/Library/Developer/Xcode/DerivedData файлы приложения
3 - нахожу в них бинарник

-----

вот файл создался после сборки проекта в xcode , и там есть файл расширения .app - это и есть нужный мне бинарник    (MeetWay.app)

![[Снимок экрана 2026-04-09 в 10.12.11 1.png]]

переместил на рабочий стол MeetWay.app

далее - нужно перейти в этот файл

`cd ~/Desktop/MeetWay.app`

ну и начинаю анализ файла через радар 2 
(вручную не получится посмотреть - так как это бинарные данные)

сперва гляну - че за файлы есть внутри моего бинарника 

`ls -la ~/Desktop/MeetWay.app/`

```bash
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % ls -la ~/Desktop/MeetWay.app/
total 124888
-rwxr-xr-x   1 evgeniy  staff     35024  9 апр 10:07 __preview.dylib
drwxr-xr-x   3 evgeniy  staff        96  7 апр 00:34 _CodeSignature
drwxr-xr-x  15 evgeniy  staff       480  9 апр 10:07 .
drwx------@ 31 evgeniy  staff       992  9 апр 10:15 ..
-rw-r--r--   1 evgeniy  staff     14412  7 апр 01:18 AppIcon60x60@2x.png
-rw-r--r--   1 evgeniy  staff     19132  7 апр 01:18 AppIcon76x76@2x~ipad.png
-rw-r--r--   1 evgeniy  staff  22615696  7 апр 01:19 Assets.car
-rw-r--r--   1 evgeniy  staff     15136  7 апр 00:25 embedded.mobileprovision
drwxr-xr-x  27 evgeniy  staff       864  7 апр 00:34 Frameworks
-rw-r--r--   1 evgeniy  staff       996  7 апр 00:25 GoogleService-Info.plist
-rw-r--r--   1 evgeniy  staff      3717  7 апр 00:26 Info.plist
-rwxr-xr-x   1 evgeniy  staff     91664  9 апр 10:07 MeetWay
-rwxr-xr-x   1 evgeniy  staff  41070784  9 апр 10:07 MeetWay.debug.dylib
-rw-r--r--   1 evgeniy  staff         8  7 апр 00:26 PkgInfo
-rw-r--r--   1 evgeniy  staff     49852  7 апр 00:25 words.txt
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```

-rwxr-xr-x   1 evgeniy  staff  41070784  9 апр 10:07 MeetWay.debug.dylib

MeetWay.debug.dylib - вот эта папка интересна для меня!


выполняю поиск 
```q
# Быстрый поиск через `strings` (ищем наши ключевые слова)
strings ./MeetWay.debug.dylib | grep -i "kkr15\|key_1_enter\|token\|password\|secret"

------------------------------------------

# Поиск вызовов `NSLog` через `radare2`
r2 -A -q -c "/ NSLog" ./MeetWay.debug.dylib 2>/dev/null

------------------------------------------

#  Поиск вызовов `os_log` и `print`
r2 -A -q -c "/ os_log" ./MeetWay.debug.dylib 2>/dev/null
r2 -A -q -c "/ print" ./MeetWay.debug.dylib 2>/dev/null

------------------------------------------

# увидеть, что вообще логируется (все строки с "log")
strings ./MeetWay.debug.dylib | grep -i "log" | head -30
```

первая же проверка нашла имена токенов

DEBUG registerAPNSToken ===
APNS Token
FCM Token:
fcmToken
password
key_1_enter
kkr15

```q
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % strings ./MeetWay.debug.dylib | grep -i "kkr15\|key_1_enter\|token\|password\|secret"
RecoverPassword
$s7MeetWay15RecoverPasswordV4bodyQrvp
kkr15
Password
notificationTokens
key_1_enter
yandexToken
 getUserDataWithTokenRefresh jwt
 getUserDataWithTokenRefresh
 DEBUG registerAPNSToken ===
 APNS Token
apnsToken
 APNS Device Token (hex):
apnsDeviceToken
 FCM Token:
fcmToken
  key_1_enter
 key_1_enter ContentView
EnterPassword
 APNS Token:
 APNS Token
SS5email_SS8passwordt
application:didRegisterForRemoteNotificationsWithDeviceToken:
initWithAccessKey:secretKey:
messaging:didReceiveRegistrationToken:
setAPNSToken:
password
yandexToken
notificationTokens
_showPassword
_password
RecoverPassword
token
_notificationToken
_password
_showPassword
_allertCountPasswordCharacters




evgeniy@Evgeniys-MacBook-Pro MeetWay.app % strings ./MeetWay.debug.dylib | grep -i "kkr15\|key_1_enter\|token\|password\|secret"
RecoverPassword
$s7MeetWay15RecoverPasswordV4bodyQrvp
kkr15
Password
notificationTokens
key_1_enter
yandexToken
 getUserDataWithTokenRefresh jwt
 getUserDataWithTokenRefresh
 DEBUG registerAPNSToken ===
 APNS Token
apnsToken
 APNS Device Token (hex):
apnsDeviceToken
 FCM Token:
fcmToken
  key_1_enter
 key_1_enter ContentView
EnterPassword
 APNS Token:
 APNS Token
SS5email_SS8passwordt
application:didRegisterForRemoteNotificationsWithDeviceToken:
initWithAccessKey:secretKey:
messaging:didReceiveRegistrationToken:
setAPNSToken:
password
yandexToken
notificationTokens
_showPassword
_password
RecoverPassword
token
_notificationToken
_password
_showPassword
_allertCountPasswordCharacters
```


-----------


а вот так можно сделать полный фарш 

больше значений

```bash
strings ./MeetWay.debug.dylib | grep -iE "kkr15|key_1_enter|token|password|secret|key|api|auth|jwt|access|refresh|session|cookie|credit|cvv|pin|login|signin|signup|email|phone|user|pass|hash|salt|iv|nonce|cert|private|public|pem|p12|pkcs|firebase|aws|yandex|apns|fcm|notification"
```

отчет по найденому - находится ниже этих логов

```c
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % r2 -A -q -c "/ NSLog" ./MeetWay.debug.dylib 2>/dev/null
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % strings ./MeetWay.debug.dylib | grep -iE "kkr15|key_1_enter|token|password|secret|key|api|auth|jwt|access|refresh|session|cookie|credit|cvv|pin|login|signin|signup|email|phone|user|pass|hash|salt|iv|nonce|cert|private|public|pem|p12|pkcs|firebase|aws|yandex|apns|fcm|notification"
UserDataProfile
UserAuth
UserDataCellsInfo
$s7MeetWay8GoalCellV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
CodingKeys
LocatesUser
$s7MeetWay11LocatesUserV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
$s7MeetWay12LocationInfoV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
$s7MeetWay5placeV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
postAchivementsData
$s7MeetWay19postAchivementsDataV10CodingKeys33_30F118A04066D921D14E82012D529912LLO
KeychainManager
KeychainHelper
PHAuthorizationStatus
OpenExternalURLOptionsKey
NUIApplicationOpenExternalURLOptionsKey
FileAttributeKey
NNSFileAttributeKey
URLResourceKey
NNSURLResourceKey
FirstScreenUserView
$s7MeetWay19FirstScreenUserViewV4bodyQrvp
UNAuthorizationOptions
CameraPreviewUIView
UniversalVideoPlayerUIView
SetActiveOptions
NAVAudioSessionSetActiveOptions
NAVAudioSessionCategoryOptions
FireBaseManager
User2
RecoverPassword
$s7MeetWay15RecoverPasswordV4bodyQrvp
RoadMapsUserView
$s7MeetWay16RoadMapsUserViewV4bodyQrvp
UserRowView
$s7MeetWay11UserRowViewV4bodyQrvp
AuthButtonStyle
$s7MeetWay15AuthButtonStyleV8makeBody13configurationQr7SwiftUI0dE13ConfigurationV_tF
PinSessionManager
ActiveTab
UNAuthorizationStatus
ObjectStorageYandex
NAVAudioSessionInterruptionOptions
NAVAudioSessionInterruptionType
YandexDBManager
User2
YandexUserInfo
YandexAuthResponse
MessageAnchorKey
MessageFramePreferenceKey
VideoPlayerUIView
$s7MeetWay5ChatsV15MessageItemViewV011mainContentF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV012videoAndTextdF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV05videodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser8showTextQrSb_SbtF
$s7MeetWay5ChatsV15MessageItemViewV05audiodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV05photodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser8showTextQrSb_SbtF
$s7MeetWay5ChatsV15MessageItemViewV013photoAndAudiodF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUser8showTextQrSb_SbtF
$s7MeetWay5ChatsV15MessageItemViewV08allThreedF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV016reactionsOverlayF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV012audioAndTextdF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV04textdF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
$s7MeetWay5ChatsV15MessageItemViewV013messageStatusF033_58B9898145204CA89E0CDD64611B323BLL13isCurrentUserQrSb_tF
NNSAttributedStringKey
NSKeyValueObservingOptions
AVKeyValueStatus
NotificationService
UNNotificationSetting
PlasesUserView
$s7MeetWay14PlasesUserViewV4bodyQrvp
YandexLoginObserver
AuthResponse
$s7MeetWay12RegistrationV12AuthResponseV10CodingKeys33_13EB04D5D94B59450665E9733D40AF31LLO
CodingKeys
UserData
$s7MeetWay12RegistrationV8UserDataV10CodingKeys33_13EB04D5D94B59450665E9733D40AF31LLO
AllCountryScreenUserView
$s7MeetWay24AllCountryScreenUserViewV4bodyQrvp
GoalScreenUserView
$s7MeetWay18GoalScreenUserViewV4bodyQrvp
AAIVAApp
$s7MeetWay8AAIVAAppV4bodyQrvp
UNNotificationPresentationOptions
OpenURLOptionsKey
NUIApplicationOpenURLOptionsKey
UNNotificationCategoryOptions
UNNotificationActionOptions
LaunchOptionsKey
NUIApplicationLaunchOptionsKey
https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg
Unexpectedly found nil while implicitly unwrapping an Optional value
Unexpectedly found nil while unwrapping an Optional value
kkr15
Not enough bits to represent the passed value
UnsafeBufferPointer with negative count
, yandexId =
 yandexId
USER_NOT_REGISTERED
com.ivaaro.chats
messageEncryptionKey
 Keychain:
 Keychain
 Keychain,
 Keychain:
 Keychain SAVE
 Keychain SAVE:
Keychain
 Keychain SAVE:
 Keychain UPDATE:
 Keychain UPDATE:
 Keychain GET
 Keychain GET:
 Keychain GET:
 Keychain DELETE
 Keychain DELETE:
 Keychain DELETE:
_TtC7MeetWay15KeychainManager
_TtC7MeetWay14KeychainHelper
postAchivementsData
 postAchivementsData
 postAchivementsData
processedNotifications_v2
Password
EnterEmail
Email
v16@?0@"NSNotification"8
SilentPushReceived
 SEETT CALLED with yandex_id:
 JWT
 JWT
 Keychain!
 udateSearchKeys
 USER_PROFILE_INFO...
Keys=
myKeys=
anotherKeys=
Division results in an overflow
Division by zero
notificationTokens
_passwRegistration
_alertShowMessageForAllUsers
userInfoRequests
_ratingUser
_chatUsersInfo
_allUsersInfo
_keysSearch
_forceRefresh
_userProfile
_userUID
_userLocalesInfo
_arIDusers
_idUserhats
_userProfileAlien
_userLocalesInfoAlien
maxStoredNotifications
notificationsKey
: keysSearch
 keysSearch
: LocatesUser
 LocatesUser
 USER_PROFILE_INFO
 USER_PROFILE_INFO
 USER_PROFILE_INFO
com.ivaaro.mediacache
Failed to create export session
activeDownloads
USERS
users
/Users/evgeniy/Desktop/
/IVAARO1 2/IVAARO/MANAGER_S/VideoCompanents.swift
 CameraPreviewUIView:
 CameraPreviewUIView:
 CameraPreviewUIView layoutSubviews: bounds =
 AVAudioSession:
 AVAudioSession
 AVAudioSession:
   - UserInfo:
 AVAudioSession
 AVAudioSession
 AVAudioSession
 setActive: isInputAvailable =
 setActive: isOtherAudioPlaying =
 setActive: isInputAvailable =
 setActive: isOtherAudioPlaying =
 AVAudioSession
 AVAudioSession
   - isActive:
 AVAudioSession
 UniversalVideoPlayerUIView: setupPlayer
session
_TtC7MeetWay19CameraPreviewUIView
audioSessionConfiguredForRecording
activeInput
_TtC7MeetWay26UniversalVideoPlayerUIView
   - AVAudioSession
   - AVAudioSession
 AVAudioSession
 AVAudioSession
 Capture session
 isSessionReady = true
AVPlayerItemDidPlayToEndTimeNotification
 isSessionReady =
Email
 email.
statusUser
allidComplaintUsers
localeUserAll
search_keywords
profileInfo.allidComplaintUsers
 userId!
idUserBADviolator
userGOOGpersonSender
Invalid userId
Auth
_TtC7MeetWay15FireBaseManager
 / updateUserData  Firestore.firestore
 updateUserData FireBaseManager
  FireBaseManager =
MeetWay/FireBaseManager.swift
: - jsonCellString= FireBaseManager3
: - locatesAllJson= FireBaseManager
 FireBaseManager =
 / registrNewUser FireBaseManager
 registrNewUser: FireBaseManager
 Remote: Change playback position command received
 Remote: Skip backward command received
 Remote: Skip forward command received
 Remote: Toggle play/pause command received
 Remote: Pause command received
 Remote: Play command received
 email
internalJWT
current_pin_session
_TtC7MeetWay17PinSessionManager
v16@?0@"UNNotificationSettings"8
hasRequestedNotifications
dateLastShowAlertForAllUsers
pink
key_1_enter
: postAchivementsData =
 postAchivementsData =
baket-ivaaro
users/
MeetWay/ObjectStorageYandex.swift
v24@?0@"AWSS3PutObjectOutput"8@"NSError"16
v24@?0@"AWSS3GetObjectOutput"8@"NSError"16
v24@?0@"AWSS3DeleteObjectOutput"8@"NSError"16
 User ID:
v24@?0@"AWSS3HeadObjectOutput"8@"NSError"16
_TtC7MeetWay19ObjectStorageYandex
https://storage.yandexcloud.net/
https://storage.yandexcloud.net/baket-ivaaro/
https://functions.yandexcloud.net/d4efavboiige7c7leqkf
https://cns.api.ilb.cloud.yandex.net
arn:aws:sns::b1ggdrk2tndoa2t1f6jf:app/APNS_SANDBOX/testSandboxPush
 JWT
 JWT
 Keychain,
 JWT
 JWT
 JWT
yandexToken
userData
 JWT
 JWT
 getValidJWT:
 JWT
 Keychain
 JWT
 JWT
 getUserDataWithTokenRefresh jwt
 getUserDataWithTokenRefresh
 yandexId:
 isUserLoggedIn jwt
 logout jwt
 getCurrentJWT jwt
 didFinishLogin jwt
https://login.yandex.ru/info?format=json
MeetWay/YandexDBManager.swift
OAuth
Authorization
default_email
searchKeywords
 JWT
 DEBUG getUserData ===
 yandexId:
 JWT
 JWT
getUserData
yandexId
locale_user_all
 locale_user_all
 DEBUG LOCALE_USER_ALL ===
 locale_user_all
 locale_user_all:
 locale_user_all
active
 JWT
 DEBUG updateUserDataProfileInfo ===
   - email:
   - statusUser:
   - statusUser
 statusUser
 statusUser
 DEBUG updateUserDataLocatesUser ===
 DEBUG updateUserDataCellAllInfo ===
 DEBUG updateUserDatakeysSearch ===
 DEBUG getUserInfo ===
 DEBUG searchUsersByMultipleKeywords ===
yandex_id
 DEBUG deleteUserAccount ===
 DEBUG handleUserReaction ===
handleUserReaction
 DEBUG findPostsByAuthor ===
 markMessagesAsRead jwt
   - receiverId:
   - currentUserId:
 addReactionToEncryptedMessage jwt
: JWT
 JWT
getUserChats
 DEBUG registerAPNSToken ===
 APNS Token
 JWT
_TtC7MeetWay15YandexDBManager
onLoginResult
jwtMemoryCache
: JWT
AuthError
 JWT
updateUserComplaint
 JWT
apnsToken
No data received
getUserChatsWithPreview
: JWT
 JWT
receiverId
 JWT:
   - JWT
 JWT
 JWT
userId
   - userId:
   - JWT (
   - yandex_id:
findPostsByAuthor
authorId
userAction
searchUsers
keywords
 users
updateUserActions
: JWT
updateUserRating
updateUserSearchKeywords
updateUserCells
updateUserLocations
updateUserProfile
Failed to get user data
 ( callGetUserDataFunction ):
 User data
User data not found in response
 User data
 userData:
 JWT
 JWT
 JSON JWT
Authentication failed
 'jwt'
 JWT
 JWT
 Keychain
 JWT
 Keychain
 (JWT):
 JWT:
Failed to save JWT
JWT not found in response
USER_NOT_REGISTERED:
No data received from server
 API
 JWT
 JWT
User already
 USER_NOT_REGISTERED
 USER_NOT_REGISTERED
 JWT
 'jwt'
 JWT
login
_TtCVV7MeetWay5Chats15MessageItemView17VideoPlayerUIView
/Users/evgeniy/Desktop/
/IVAARO1 2/IVAARO/View/searchScreen+chats/chats.swift
 YandexDBManager
   - userInfo:
activeReactionPickerMessageId =
themeColorUser
tap - pink
otherUserID
notificationType
_TtC7MeetWay19NotificationService
   - otherUserId:
user
email
_TtC7MeetWay19YandexLoginObserver
 Firebase
 YandexLoginSDK:
 APNS
 APNS Device Token (hex):
apnsDeviceToken
 APNS:
 FCM Token:
fcmToken
  key_1_enter
https://storage.yandexcloud.net
MeetWay/AAIVAApp.swift
Division results in an overflow in remainder operation
Division by zero in remainder operation
 key_1_enter ContentView
EnterPassword
OpenChatNotification
 APNS Token:
 APNS Token
3d-open-toolbox-with-wrench-screwdriver-peeking-out-3d-illustration (1)
hand-is-writing-white-keyboard-with-blue-pen
vivid-blurred-colorful-wallpaper-background
$s7SwiftUI19UIViewRepresentableP
yAwSA49_GGG_AEyADyACyAH_AEyADyACyAEyAEyAHA4_GAOG_AEyAFy
$ss21_ObjectiveCBridgeableP
Si8newCount_SS10userActionSDyS2SG9reactionst
G5users_SSSg12lastDocumentt
So20AVAssetExportSessionC
SaySo29AVAudioSessionPortDescriptionCG
So20AVAssetExportSessionCSg
So6UIViewC
So16AVCaptureSessionC
So16AVCaptureSessionCSg
G5users_So19FIRDocumentSnapshotCSg12lastDocumentt
SS5email_SS8passwordt
yAwS
So22UNNotificationSettingsC
yAWSgGG
yAAyAiVG_Qo__
yAByAiVG_Qo__
G5users_SSSg12lastDocumentt
ySi8newCount_SS10userActionSDyS2SG9reactionst
$s7SwiftUI13PreferenceKeyP
$s7SwiftUI29UIViewControllerRepresentableP
yAAyAAyA11_yADyAAyAiVGAAy
yAByAByA11_yADyAByAiVGABy
GAkEy
setBool:forKey:
setKey:
application:continueUserActivity:restorationHandler:
application:didDiscardSceneSessions:
activeFormat
addNotificationRequest:withCompletionHandler:
removeAllDeliveredNotifications
application:configurationForConnectingSceneSession:options:
application:didFailToContinueUserActivityWithType:error:
application:didFailToRegisterForRemoteNotificationsWithError:
application:didReceiveLocalNotification:
application:didReceiveRemoteNotification:
application:didReceiveRemoteNotification:fetchCompletionHandler:
application:didRegisterForRemoteNotificationsWithDeviceToken:
application:didRegisterUserNotificationSettings:
application:didUpdateUserActivity:
application:handleActionWithIdentifier:forLocalNotification:completionHandler:
application:handleActionWithIdentifier:forLocalNotification:withResponseInfo:completionHandler:
application:handleActionWithIdentifier:forRemoteNotification:completionHandler:
application:handleActionWithIdentifier:forRemoteNotification:withResponseInfo:completionHandler:
application:handleEventsForBackgroundURLSession:completionHandler:
application:userDidAcceptCloudKitShareWithMetadata:
application:willContinueUserActivityWithType:
applicationDidBecomeActive:
applicationDidReceiveMemoryWarning:
applicationShouldAutomaticallyLocalizeKeyCommands:
applicationShouldRequestHealthAuthorization:
applicationWillResignActive:
arrayForKey:
authorizationStatus
authorizationStatusForAccessLevel:
boolForKey:
captureOutput:didPauseRecordingToOutputFileAtURL:fromConnections:
contentsOfDirectoryAtURL:includingPropertiesForKeys:options:error:
currentNotificationCenter
dictionaryForKey:
doubleForKey:
getNotificationSettingsWithCompletionHandler:
handleAudioInterruptionWithNotification:
hash
initWithAccessKey:secretKey:
initWithDomain:code:userInfo:
initWithSession:
isActive
loadValuesAsynchronouslyForKeys:completionHandler:
messaging:didReceiveRegistrationToken:
notification
objectForInfoDictionaryKey:
postNotificationName:object:
postNotificationName:object:userInfo:
registerForRemoteNotifications
removeAllPendingNotificationRequests
removeObjectForKey:
requestAccessForMediaType:completionHandler:
requestAuthorization:
requestAuthorizationForAccessLevel:handler:
requestAuthorizationWithOptions:completionHandler:
setAPNSToken:
setActive:withOptions:error:
setDouble:forKey:
setNotificationCategories:
setObject:forKey:
setSessionPreset:
setUserInfo:
sharedSession
standardUserDefaults
startSessionAtSourceTime:
statusOfValueForKey:error:
stringForKey:
userInfo
userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler:
userNotificationCenter:openSettingsForNotification:
userNotificationCenter:willPresentNotification:withCompletionHandler:
UIViewType
email
statusUser
allidComplaintUsers
active
password
yandexID
yandexToken
_showCardGoalVSachivements
_redactCardGoalVSachivements
_alertDeleteCardGoalVSachivements
_tapShowAllMyPostsGoalsAndAchivements
_idUserBADviolator
_userGOOGpersonSender
_ObjectiveCType
notificationTokens
_passwRegistration
_alertShowMessageForAllUsers
userInfoRequests
_ratingUser
_chatUsersInfo
_allUsersInfo
_keysSearch
_forceRefresh
_userProfile
_userUID
_userLocalesInfo
_arIDusers
_idUserhats
_userProfileAlien
_userLocalesInfoAlien
maxStoredNotifications
notificationsKey
activeDownloads
_ObjectiveCType
_textStatusUser
_showInfoUser
_idUserBADviolator
_userGOOGpersonSender
UIViewType
session
_isSessionReady
audioSessionConfiguredForRecording
activeInput
_ObjectiveCType
email
relevantOFmyKeys
countRelevantToMyKeys
_email
_EmailFocused
countRelevantToMyKeys
totalKeys
_showPassword
_email
_password
_EmailFocused
RecoverPassword
FirstScreenUserView
GoalScreenUserView
RoadMapsUserView
AllCountryScreenUserView
PlasesUserView
_nativeLanguage
_nativeLanguageFocused
_textStatusUser
_activeTab
_showInfoUser
_redactCardGoalVSachivements
_alertDeleteCardGoalVSachivements
_idUserBADviolator
_userGOOGpersonSender
onLoginResult
jwtMemoryCache
email
relevantOFmyKeys
countRelevantToMyKeys
token
UIViewType
UIViewControllerType
_activeReactionPickerMessageId
_showMicrophonePermissionAlert
_isRefreshing
_notificationToken
_countryUser
_languageUser
_professionUser
_nameUser
_avatarURLUser
_userID
_currentUserId
_ObjectiveCType
_countryLiveFocused
otherUserId
userInfo
onLoginResult
_email
_password
_showPassword
_yandexObserver
_allertCountPasswordCharacters
user
login
email
_hasCheckedAuth
_ObjectiveCType
UNUserNotificationCenterDelegate
@"UIView"24@0:8@"UIScrollView"16
v32@0:8@"UIScrollView"16@"UIView"24
v40@0:8@"UIScrollView"16@"UIView"24d32
v32@0:8@"UIApplication"16@"UIUserNotificationSettings"24
v32@0:8@"UIApplication"16@"UILocalNotification"24
v48@0:8@"UIApplication"16@"NSString"24@"UILocalNotification"32@?<v@?>40
v56@0:8@"UIApplication"16@"NSString"24@"UILocalNotification"32@"NSDictionary"40@?<v@?>48
@"UIViewController"40@0:8@"UIApplication"16@"NSArray"24@"NSCoder"32
B40@0:8@"UIApplication"16@"NSUserActivity"24@?<v@?@"NSArray">32
v32@0:8@"UIApplication"16@"NSUserActivity"24
@"UISceneConfiguration"40@0:8@"UIApplication"16@"UISceneSession"24@"UISceneConnectionOptions"32
v40@0:8@"UNUserNotificationCenter"16@"UNNotification"24@?<v@?Q>32
v40@0:8@"UNUserNotificationCenter"16@"UNNotificationResponse"24@?<v@?>32
v32@0:8@"UNUserNotificationCenter"16@"UNNotification"24
evgeniy@Evgeniys-MacBook-Pro MeetWay.app %
```



сами значения скрывает система от меня , но если динамически просмотреть логи в реальном времени используя  ==idevicesyslog==, то в логах из idevicesyslog можно будет найти через поиск все эти значения , имена переменных, что я нашел в этом тесте, и также возможно, эти значения попадают в бекап





**Вывод:** ❌ **Тест НЕ ПРОЙДЕН (FAIL)**

что тут интересного :


 AWS / Yandex Cloud

```http
initWithAccessKey:secretKey:

https://storage.yandexcloud.net/
arn:aws:sns::b1ggdrk2tndoa2t1f6jf:app/APNS_SANDBOX/testSandboxPush
```

захардкоженные URL и пути

```http

https://storage.yandexcloud.net/baket-ivaaro/users/ShWOosv5KHeuyvGlBnrGInRGzFv2/1EEB32E4-2BC3-4A6D-9B73-4FD4869A0E9D.jpg

https://functions.yandexcloud.net/d4efavboiige7c7leqkf
/Users/evgeniy/Desktop/IVAARO1 2/IVAARO/MANAGER_S/VideoCompanents.swift
```

Ключи и токены

```q
messageEncryptionKey
kkr15
key_1_enter
yandexToken
apnsToken
fcmToken
internalJWT
jwtMemoryCache
```

Keychain операции (доказательство, что разработчик пытался что-то спрятать, но оставил отладку)

```c
Keychain SAVE
Keychain UPDATE:
Keychain GET
Keychain DELETE
```

Логирование всего подряд

```q
DEBUG registerAPNSToken ===
DEBUG getUserData ===
DEBUG updateUserDataProfileInfo ===
DEBUG deleteUserAccount ===
DEBUG handleUserReaction ===
DEBUG findPostsByAuthor ===
DEBUG registerAPNSToken ===
```

---------


# теперь самое интересное!
буду в бинарнике MeetWay.debug.dylib искать сами коючи , значения токенов итд

открываю папку MeetWay.app

открываю в ней бинарник `r2 ./MeetWay.debug.dylib`

даю команду aaaa  (это для полного сканирования)
вот так:
 и жду 
 
![[Снимок экрана 2026-04-09 в 10.50.21.png]]

вот результат анализа бинарника
```q
[0x00004000]> aaaa
INFO: Analyze all flags starting with sym. and entry0 (aa)
INFO: Analyze imports (af@@@i)
INFO: Analyze symbols (af@@@s)
INFO: Analyze all functions arguments/locals (afva@@F)
INFO: Analyze function calls (aac)
INFO: Analyze len bytes of instructions for references (aar)
INFO: Check for objc references (aao)
INFO: Parsing metadata in ObjC to find hidden xrefs
INFO: Found 0 objc xrefs in 481 dwords
INFO: Finding and parsing C++ vtables (avrr)
INFO: Analyzing methods (af @@ method.*)
INFO: Finding function preludes (aap)
INFO: Emulate functions to find computed references (aaef)
INFO: Recovering local variables (afva@@@F)
INFO: Type matching analysis for all functions (aaft)
INFO: Propagate noreturn information (aanr)
INFO: Scanning for strings constructed in code (/azs)
INFO: Enable types.constraint for experimental type propagation
INFO: Running plugin post-analysis hooks
INFO: Finding xrefs in noncode sections (e anal.in=io.maps.x; aav)
WARN: Skipping aav because base address is zero. Use -B 0x800000 or aav0
INFO: aav: 0x00a34000-0x00a371b8 in 0xa34000-0xa371b8
INFO: aav: 0x00a34000-0x00a371b8 in 0xa53250-0xa546b0
INFO: aav: 0x00a34000-0x00a371b8 in 0xa546b0-0xa647f0
INFO: aav: 0x00a34000-0x00a371b8 in 0xa647f0-0xa6e440
INFO: aav: 0x00a34000-0x00a371b8 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a371b8-0x00a4cde8 in 0xa34000-0xa371b8
INFO: aav: 0x00a371b8-0x00a4cde8 in 0xa371b8-0xa4cde8
INFO: aav: 0x00a4cde8-0x00a4cec0 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a4cec0-0x00a4cf60 in 0xa34000-0xa371b8
INFO: aav: 0x00a4cec0-0x00a4cf60 in 0xa546b0-0xa647f0
INFO: aav: 0x00a4cec0-0x00a4cf60 in 0xa647f0-0xa6e440
INFO: aav: 0x00a4cec0-0x00a4cf60 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a4cf60-0x00a4cf68 in 0xa34000-0xa371b8
INFO: aav: 0x00a4cf60-0x00a4cf68 in 0xa371b8-0xa4cde8
INFO: aav: 0x00a4cf60-0x00a4cf68 in 0xa4cde8-0xa4cec0
INFO: aav: 0x00a4cf60-0x00a4cf68 in 0xa4cec0-0xa4cf60
INFO: aav: 0x00a4cfb8-0x00a4d298 in 0xa53250-0xa546b0
INFO: aav: 0x00a4cfb8-0x00a4d298 in 0xa546b0-0xa647f0
INFO: aav: 0x00a4cfb8-0x00a4d298 in 0xa647f0-0xa6e440
INFO: aav: 0x00a4cfb8-0x00a4d298 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a50000-0x00a52348 in 0xa4cf60-0xa4cf68
INFO: aav: 0x00a52348-0x00a53250 in 0xa50000-0xa52348
INFO: aav: 0x00a52348-0x00a53250 in 0xa52348-0xa53250
INFO: aav: 0x00a52348-0x00a53250 in 0xa53250-0xa546b0
INFO: aav: 0x00a53250-0x00a546b0 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa34000-0xa371b8
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa371b8-0xa4cde8
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa4cde8-0xa4cec0
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa4cec0-0xa4cf60
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa4cf60-0xa4cf68
INFO: aav: 0x00a546b0-0x00a647f0 in 0xa4cf68-0xa4cfb8
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa50000-0xa52348
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa52348-0xa53250
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa53250-0xa546b0
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa546b0-0xa647f0
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa647f0-0xa6e440
INFO: aav: 0x00a647f0-0x00a6e440 in 0xa6e440-0xa6ea00
INFO: aav: 0x00a6e440-0x00a6ea00 in 0xa34000-0xa371b8
INFO: aav: 0x00a6e440-0x00a6ea00 in 0xa53250-0xa546b0
INFO: aav: 0x00a6e440-0x00a6ea00 in 0xa546b0-0xa647f0
INFO: aav: 0x00a6e440-0x00a6ea00 in 0xa647f0-0xa6e440
INFO: aav: 0x00a6e440-0x00a6ea00 in 0xa6e440-0xa6ea00
[0x00004000]>
```

далее тоже начинаю искать все те токены, что находил я ранее 
/ kkr15
/ YCAJESl8
/ YCMZYgs2
/ messageEncryptionKey
/ 8137tr8gf

вот так:

```c
[0x00004000]> / kkr15
0x0090f9aa hit4_0 . Optional valuekkr15vkewjrbg@#$%$**.

[0x00004000]> / YCAJESl8
0x0092be10 hit5_0 .rYCAJESl8Vc6eod-TSiavKAGV.

[0x00004000]> / YCMZYgs2
0x0092be30 hit6_0 .SiavKAGVhYCMZYgs2XIJupQenfJbRsqeC.

[0x00004000]> / messageEncryptionKey
0x0090fee0 hit7_0 .messageEncryptionKey .

[0x00004000]> / 8137tr8gf
0x0091d968 hit8_0 . 8137tr8gf.
[0x00004000]> registerAPNSToken

[0x00004000]> /registerAPNSToken
| /re [addr]  search references using esil

[0x00004000]> / registerAPNSToken
0x00a99e19 hit9_0 .SDySSypGG_tFregisterAPNSToken_10completionySS.
0x00d1138a hit9_1 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x00d113d8 hit9_2 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x01e1e135 hit9_3 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x01e1e18a hit9_4 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x01e2002c hit9_5 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x01e2121a hit9_6 .ndexDBManagerC17registerAPNSToken_10completionySS.
0x009221df hit9_7 .===  DEBUG registerAPNSToken ===.

[0x00004000]> / fcmToken
0x0092b951 hit10_0 . FCM Token: fcmToken Push.

[0x00004000]> / apnsToken
0x00922e93 hit11_0 .registerDeviceapnsTokencnsChannelArn.

[0x00004000]> / jwtMemoryCache
0x00a99a94 hit12_0 .LoggedInSbyFjwtMemoryCache33_7108AB7FE2C72.
0x00d10d4f hit12_1 .ndexDBManagerC14jwtMemoryCache33_7108AB7FE2C72.
0x00d10da8 hit12_2 .ndexDBManagerC14jwtMemoryCache33_7108AB7FE2C72.
0x025e7891 hit12_9 .ndexDBManagerC14jwtMemoryCache33_7108AB7FE2C72.
0x009226d4 hit12_10 .LonLoginResultjwtMemoryCachecacheExpirycac.
0x00a19da9 hit12_11 .LonLoginResultjwtMemoryCachecacheExpirycac.

[0x00004000]> / internalJWT
0x0091c456 hit13_0 .: internalJWTmyID.

[0x00004000]> / yandexToken
0x00a99601 hit14_0 .JWTToken11yandexToken8userData10compl.
0x00a99f41 hit14_1 .2864048A0FEFLL11yandexToken8userData10compl.
0x00ab536b hit14_2 .n11yandexTokenSSSgv4name.
0x00d1032b hit14_3 .C11getJWTToken11yandexToken8userData10compl.
0x00d10390 hit14_4 .C11getJWTToken11yandexToken8userData10compl.
0x00d1155f hit14_5 .2864048A0FEFLL11yandexToken8userData10compl.
0x00d4028f hit14_6 .tWay8UserAuthV11yandexTokenSSSgvg_$s7MeetW.
0x00d402b8 hit14_7 .tWay8UserAuthV11yandexTokenSSSgvpMV_$s7Mee.
0x01e18863 hit14_8 .2864048A0FEFLL11yandexToken8userData10compl.
0x01e188ff hit14_9 .2864048A0FEFLL11yandexToken8userData10compl.
0x01e1899c hit14_10 .2864048A0FEFLL11yandexToken8userData10compl.
0x01e18a68 hit14_11 .2864048A0FEFLL11yandexToken8userData10compl.
0x01e24786 hit14_30 .2864048A0FEFLL11yandexToken8userData10compl.
0x01e24891 hit14_31 .2864048A0FEFLL11yandexToken8userData10compl.
0x02377b5d hit14_32 .2864048A0FEFLL11yandexToken8userData10compl.
0x0091ffde hit14_33 .onContent-TypeyandexTokenuserData.
0x00a16a7b hit14_34 .sswordyandexIDyandexTokenarrayA.
[0x00004000]>
```


разбор первой строчки

```c

[0x00004000]> / kkr15 -это мой запрос


0x0090f9aa hit4_0 . Optional valuekkr15vkewjrbg@#$%$**. - это ответ радара

и радар вернул адрес памяти 0x0090f9aa где и хранится значение найденное!

```

#### теперь  я могу достать это значение
 
 вывести строку по адресу

psz 0x0090f9aa


делаю запрос / ответ
```q
[0x00004000]> psz 0x0090f9aa
\xff\x83
[0x00004000]>
```

`radare2` показывает `\xff\x83`, потому что по этому адресу не строка, а ссылка (указатель) на реальную строку

это не строка, а часть **указателя** (ссылки на реальные данные). Там нечего читат



---

или можно сделать так!
```c
evgeniy@Evgeniys-MacBook-Pro MeetWay.app % strings -t x ./MeetWay.debug.dylib | grep -E "8137tr8gf|kkr15|YCAJESl8|YCMZYgs2|messageEncryptionKey"

90f9aa kkr15
90fee0 messageEncryptionKey
91d968 8137tr8gf
92be10 YCAJESl8Vc6eod-TSiavKAGVh
92be30 YCMZYgs2XIJupQenfJbRsqeCNLN5xjaKtGj17ep2

```



## ВЫВОДЫ

```q
 Результаты статического анализа (MASTG-TEST-0297)

С помощью команды strings -t x в бинарнике MeetWay.debug.dylib обнаружены следующие чувствительные данные в открытом виде:

| Адрес    | Значение      | Тип |


| 0x91d968 | `8137tr8gf` | Соль AES (значение переменной `kkr15`) |

| 0x92be10 | `YCAJESl8Vc6eod-TSiavKAGVh` | AWS Access Key |

| 0x92be30 | `YCMZYgs2XIJupQenfJbRsqeCNLN5xjaKtGj17ep2` | AWS Secret Key |

| 0x90fee0 | `messageEncryptionKey` | Ключ шифрования сообщений |

Вывод: ❌ Тест не пройден. Приложение содержит хардкод криптографических ключей и AWS-секретов
```


