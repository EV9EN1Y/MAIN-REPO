https://mas.owasp.org/MASTG/tests/android/MASVS-PRIVACY/MASTG-TEST-0318/
# MASTG-TEST-0318: References to SDK APIs Known to Handle Sensitive User Data

не использует ли приложение сторонние SDK (библиотеки), которые собирают пользовательские данные, но при этом разработчик мог забыть указать это в политике конфиденциальности

нужно статически искать в коде методы и библиотеки - которые могут собирать данные юзеров, 
и нужно их все найти, а затем свериться с политикой конфиденциальности 
и если данные реально отправляются - но в политике не указаны - это может считаться нарушением!

То есть СДК которые используются в коде приложения, они все должны быть указаны в политике!

вот тут я проверял в динамике то - ухдят ли данные персональные, которые не указаны в политике
[[P_dast-burp_данные-не-указанные-в-политике]]

а в текущем тесте 0318 - я проверю, указаны ли в политике все SDK которые есть в коде!


Политика должна содержать информацию о том, какие SDK используются, какая компания-разработчик, какие типы пользовательских данных они собирают и для каких целей [](https://developer.huawei.com/consumer/cn/doc/appgallery-connect-Guides/agc-auth-server-java-sdksecurity2-0000001714188649)[](https://support.google.com/googleplay/android-developer/answer/10144311?visit_id=638703273245044890-4200183755&rd=1). Эта информация также должна быть отражена в разделе **«Безопасность данных» (Data Safety)** в Google Play Console

------

##  ### ClassyShark - спец прога которая проанализирует зависимости и выдаст готовый список
и не только )

скачиваю край версию 
https://github.com/google/android-classyshark/releases/download/8.2/ClassyShark.jar


запуск
```c
иду в папку
cd /Users/evgeniy/Desktop/ClassyShark


запускаю 

java -jar ClassyShark.jar -export /Users/evgeniy/Desktop/meetway.apk
```

и теперь в родной дирректории ClassyShark лежат файлы отчета

и там есть файл   all_classes.txt -  в нем результаты отчета
но там тысячи строк, теперь задача - выудить из них библиотеки
но это ерунда способ... так как невозможно проверить все зависомости...
с таким же "успехом" можно было бы искать и через JADX

```q
grep -iE "firebase|googleanalytics|google-ads|google-analytics|googleads|google\.ads|facebook|appsflyer|adjust|yandex|crashlytics|mixpanel|amplitude|kochava|branch|clevertap|mparticle|singular|tenjin|flurry|countly|bugly|tbs|qq|wechat|alipay|umeng|youmeng|gson|okhttp|retrofit|glide|picasso|fresco|butterknife|dagger|rxjava|rxandroid|eventbus|greenrobot|ormlite|realm|room|sqlite|leakcanary|timber|stetho|chuck|okio|kotlinx|coroutines|androidx|support-v4|appcompat|recyclerview|cardview|material|design|constraintlayout|navigation|paging|viewpager|exoplayer|vlc|ijkplayer|ffmpeg|webview|chromium|tbs|x5|tencent|huawei|xiaomi|oppo|vivo|oneplus|samsung|amazon|firebase-perf|firebase-crash|firebase-auth|firebase-database|firebase-storage|firebase-messaging|firebase-config|firebase-inappmessaging|firebase-dynamiclinks|firebase-installations|firebase-analytics-ktx|firebase-crashlytics-ndk|google-play-services|gms|location|maps|places|fusedlocation|geofencing|nearby|awareness|mlkit|vision|barcode|face|textrecognition|translate|cloud|speech|text-to-speech|tensorflow|lite|camera|camerax|mediapipe|opencv|tesseract|leakcanary|blockcanary|matrix|sentry|newrelic|datadog|dynatrace|instabug|zego|agora|twilio|sinch|zoom|jitsi|webengage|customerio|intercom|zendesk|helpshift|freshchat|appboy|braze|urbanairship|pushwoosh|onesignal|airbrake|rollbar|hockeyapp|appcenter|codepush|microblink|scanbot|abbyy|ibm|sap|oracle|microsoft|apple|apache|spring|hibernate|freemarker|velocity|log4j|slf4j|junit|testng|mockito|robolectric|espresso|uiautomator|selenium|appium|cucumber|gherkin|jacoco|sonarqube|jenkins|gradle|maven|ant|groovy|kotlin|java|scala|clojure|perl|python|ruby|php|javascript|typescript|nodejs|react-native|flutter|dart|unity|unreal|cocos|libgdx|lwjgl|jmonkey|andengine|spritekit|scenekit|metal|opengl|vulkan|directx" all_classes.txt
```
короче - метод = фигня


-------

план Б

## **LibScout**  - он сканирует APK и сравнивает его с базой профилей известных библиотек, чтобы найти совпадения даже в обфусцированном коде


ставлю

```q
cd /Users/evgeniy/Desktop/LIB

# Скачиваем готовый JAR (взято из проверенного источника)
curl -L -o LibScout.jar https://github.com/reddr/LibScout/releases/download/v1.0.2/LibScout.jar

качаю базу профилей
git clone https://github.com/reddr/LibScout-Profiles.git

запуск анализа
java -jar LibScout.jar -o match -p LibScout-Profiles/profiles -a /путь/к/android.jar /Users/evgeniy/Desktop/meetway.apk


```
не хочет скачиваться ...


------

```js
  __  __       _    ____  _____       _  _    ____
 |  \/  | ___ | |__/ ___||  ___|_   _| || |  | ___|
 | |\/| |/ _ \| '_ \___ \| |_  \ \ / / || |_ |___ \
 | |  | | (_) | |_) |__) |  _|  \ V /|__   _| ___) |
 |_|  |_|\___/|_.__/____/|_|     \_/    |_|(_)____/

```

### короче, запускаю MobSF - он сам найдет все зависимости

запускаю докер
запускаю контейнер MobSF

```bash
  docker run -it --rm -p 8000:8000 \
    -v "$PWD:/home/mobsf/MobSF/upload" \
    opensecurity/mobile-security-framework-mobsf:latest
```

Флаг -v примонтирует текущую папку - кинешь туда APK и он появится в
веб-интерфейсе

потом открыл в браузере

http://localhost:8000

Логин/пароль по умолчанию: mobsf / mobsf

закидываю туда АПК файл и запускаю сканер


отчет готов

<img src="../../../assets/Снимок2026-07-2001.22.18.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />


🔥 Firebase (Google):
- Firebase Authentication - GenericIdpActivity, RecaptchaActivity
- Firebase Cloud Messaging - FirebaseInstanceIdReceiver, c2dm.permission.RECEIVE
- Firebase Remote Config (disabed, но SDK есть)
- API Key: AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEM

🔥 Яндекс:
- Yandex Cloud Functions - functions.yandexcloud.net
- Yandex Object Storage - storage.yandexcloud.net
- Yandex OAuth - oauth.yandex.ru, login.yandex.ru
- YandexAuthService, YandexAuthWebViewActivity

📦 SDK/Библиотеки:
- OkHttp + SSL Pinning - SslPinningInterceptor
- gRPC - io.grpc.* (куча классов)
- Coil - загрузка изображений
- CameraX - камера
- AndroidX (Compose, Credentials, Security Crypto)
- Google Sign-In


--------

скормлю некоторые файл кода - локальной LLM  - пусть сделает анализ всего кода и найдет все библиотеки

```q
### 🔥 Firebase / Google Cloud

┌─────────────────┬────────────────────────────────┬─────────────────────────────┐
│ SDK             │ Пакет                          │ Назначение                  │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase        │ com.google.firebase.auth       │ Аутентификация              │
│ Authentication  │                                │ пользователей (email,       │
│                 │                                │ Google, GitHub, Twitter,    │
│                 │                                │ Facebook, phone, OAuth)     │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase Cloud  │ com.google.firebase.messaging  │ Push-уведомления +          │
│ Messaging       │                                │ FirebaseInstanceIdReceiver  │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase        │ com.google.firebase.firestore  │ NoSQL база данных (основная │
│ Firestore       │                                │ БД приложения)              │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase        │ com.google.firebase.installati │ Управление инсталляциями    │
│ Installations   │ ons                            │ (FID)                       │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase App    │ com.google.firebase.appcheck   │ Защита API от               │
│ Check           │                                │ неавторизованного доступа   │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase        │ com.google.firebase.analytics. │ Аналитика (базовый SDK даже │
│ Analytics       │ connector                      │ если не используется        │
│ Connector       │                                │ напрямую)                   │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase Data   │ com.google.firebase.datatransp │ Транспорт данных для        │
│ Transport       │ ort                            │ аналитики                   │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firebase Remote │ (MobSF: disabled)              │ Remote Config SDK есть      │
│ Config          │                                │                             │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Google Sign-In  │ com.google.android.gms.auth /  │ Google авторизация          │
│                 │ com.google.android.gms.common  │                             │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Google Play     │ com.google.android.gms         │ Базовый слой (auth, tasks,  │
│ Services        │                                │ base)                       │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Google API      │ com.google.api                 │ Клиент для Google API       │
│ Client          │                                │                             │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Gson            │ com.google.gson                │ JSON сериализация           │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Google Protobuf │ com.google.protobuf            │ Protobuf (Firestore)        │
├─────────────────┼────────────────────────────────┼─────────────────────────────┤
│ Firestore gRPC  │ com.google.firestore / io.grpc │ Firestore через gRPC        │
└─────────────────┴────────────────────────────────┴─────────────────────────────┘

### ☁️ Яндекс

┌────────────────┬─────────────────────────────────────┬─────────────────────────┐
│ SDK            │ Пакет                               │ Назначение              │
├────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Yandex OAuth   │ com.evgeniy.meetway.service.YandexA │ Вход через Яндекс       │
│                │ uthService                          │                         │
├────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Yandex Cloud   │ functions.yandexcloud.net           │ Серверные функции       │
│ Functions      │                                     │                         │
├────────────────┼─────────────────────────────────────┼─────────────────────────┤
│ Yandex Object  │ storage.yandexcloud.net             │ S3-совместимое          │
│ Storage        │                                     │ хранилище (фото, видео, │
│                │                                     │ аудио)                  │
└────────────────┴─────────────────────────────────────┴─────────────────────────┘

### 🌐 Сетевые библиотеки

┌───────────────┬──────────────────────────────────┬────────┬────────────────────┐
│ Библиотека    │ Пакет                            │ Версия │ Назначение         │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ OkHttp        │ com.squareup.okhttp3             │ –      │ HTTP клиент        │
│               │                                  │        │ (основной)         │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ OkHttp        │ okhttp3.logging.HttpLoggingInter │ –      │ Логирование HTTP   │
│ Logging       │ ceptor                           │        │                    │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ Retrofit2     │ retrofit2                        │ –      │ REST клиент        │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ Retrofit Gson │ retrofit2.converter.gson         │ –      │ JSON конвертер для │
│ Converter     │                                  │        │ Retrofit           │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ Okio          │ okio                             │ –      │ I/O библиотека     │
│               │                                  │        │ (часть OkHttp)     │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ gRPC          │ io.grpc                          │ –      │ gRPC (Firestore)   │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ gRPC OkHttp   │ io.grpc.okhttp                   │ –      │ gRPC               │
│               │                                  │        │ транспортировка    │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ gRPC Protobuf │ io.grpc.protobuf                 │ –      │ Protobuf для gRPC  │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ gRPC Stub     │ io.grpc.stub                     │ –      │ Клиентские стабы   │
├───────────────┼──────────────────────────────────┼────────┼────────────────────┤
│ gRPC Android  │ io.grpc.android                  │ –      │ Android каналы     │
└───────────────┴──────────────────────────────────┴────────┴────────────────────┘

### 🖼️ UI / Медиа

┌─────────────┬──────────────────────┬───────────────┬───────────────────────────┐
│ Библиотека  │ Пакет                │ Версия        │ Назначение                │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ Coil        │ coil                 │ –             │ Image loading             │
│             │                      │               │ (Kotlin-first)            │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ Coil        │ coil.compose         │ –             │ Интеграция с Jetpack      │
│ Compose     │                      │               │ Compose                   │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ CameraX     │ androidx.camera      │ 1.4.1         │ Камера (core, camera2,    │
│             │                      │               │ lifecycle, video, view)   │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.compose.*   │ 1.9.0 / 1.7.0 │ Jetpack Compose (UI,      │
│ Compose     │                      │ / 1.3.0       │ Material, Material3,      │
│             │                      │               │ Animation)                │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.activity    │ 1.9.0         │ Activity + Compose        │
│ Activity    │                      │               │ интеграция                │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.lifecycle   │ 2.10.0        │ Lifecycle + ViewModel     │
│ Lifecycle   │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.navigation  │ 2.9.8         │ Compose Navigation        │
│ Navigation  │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.credentials │ 1.2.0-rc01    │ Credential Manager + Play │
│ Credentials │                      │               │ Services Auth             │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.biometric   │ 1.2.0-alpha05 │ Биометрия                 │
│ Biometric   │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.security    │ 1.1.0         │ EncryptedSharedPreference │
│ Security    │                      │               │ s                         │
│ Crypto      │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.datastore   │ 1.1.7         │ DataStore Preferences     │
│ DataStore   │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.browser     │ 1.10.0        │ Chrome Custom Tabs        │
│ Browser     │                      │               │                           │
├─────────────┼──────────────────────┼───────────────┼───────────────────────────┤
│ AndroidX    │ androidx.appcompat   │ 1.6.1         │ AppCompat                 │
│ AppCompat   │                      │               │                           │
└─────────────┴──────────────────────┴───────────────┴───────────────────────────┘

### 🧰 Системные / Инструменты

┌──────────────────────────────────────────┬─────────────────────────────────────┐
│ Библиотека                               │ Назначение                          │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ Kotlin Coroutines (core, android,        │ 1.9.0                               │
│ play-services)                           │                                     │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ Kotlinx                                  │ –                                   │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ PerfMark                                 │ io.perfmark                         │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ Conscrypt                                │ org.conscrypt                       │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ Bouncy Castle (JSSe)                     │ org.bouncycastle.jsse               │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ json                                     │ org.json                            │
├──────────────────────────────────────────┼─────────────────────────────────────┤
│ OkHttp Certificate Pinner                │ com.squareup.okhttp3.CertificatePin │
│                                          │ ner                                 │
└──────────────────────────────────────────┴─────────────────────────────────────┘

### 📦 Native Libraries (arm64-v8a)

┌─────────────────────────────────┬─────────────────────────────┐
│ .so файл                        │ Назначение                  │
├─────────────────────────────────┼─────────────────────────────┤
│ libdatastore_shared_counter.so  │ DataStore counter           │
├─────────────────────────────────┼─────────────────────────────┤
│ libimage_processing_util_jni.so │ Обработка изображений (JNI) │
├─────────────────────────────────┼─────────────────────────────┤
│ libsurface_util_jni.so          │ Surface utilities           │
├─────────────────────────────────┼─────────────────────────────┤
│ libandroidx.graphics.path.so    │ AndroidX Graphics Path      │
└─────────────────────────────────┴─────────────────────────────┘

────────────────────────────────────────────────────────────────────────────────

### 🚩 Что должно быть в политике конфиденциальности

1. Firebase Auth (Google) – email, данные профиля
2. Firebase Firestore – весь пользовательский контент
3. Firebase Cloud Messaging – push tokens
4. Firebase Analytics / Data Transport – аналитика
5. Firebase App Check – метаданные устройства
6. Firebase Installations – FID идентификатор
7. Яндекс OAuth – логин через Яндекс
8. Яндекс Cloud Functions – запросы к бэкенду
9. Яндекс Object Storage – хранение медиафайлов
10. Google Sign-In – Google авторизация
11. Coil – загрузка изображений
12. CameraX – доступ к камере/микрофону
13. Biometric – биометрия
14. Google Play Services – базовая инфраструктура
```

-------



далее нужно открыть политику конфиденциальности приложения и сверить ее с теми библиотеками , что я нашел!

------




для поиска зависимостей использовались LLM и мной
```c
┌─────┬──────────────────────────────┬───────────────────────────────────────────┐
│ Шаг │ Инструмент                   │ Что сделал                                │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 1   │ jadx --show-bad-code -d      │ Дизассемблировал APK в Java-код (27947    │
│     │ meetway_src meetway.apk      │ файлов)                                   │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 2   │ grep "^import " \*.java |    │ Вытащил все уникальные imports (7368      │
│     │ sort -u                      │ строк)                                    │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 3   │ awk                          │ Отфильтровал системные пакеты (java.*,    │
│     │                              │ javax.*, android.*, androidx.*) –         │
│     │                              │ остались только сторонние                 │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 4   │ unzip -l meetway.apk | grep  │ Прочитал файлы версий каждой библиотеки   │
│     │ META-INF.*.version           │                                           │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 5   │ find по класса́м Firebase     │ Определил точные подпродукты Firebase     │
│     │                              │ (Auth, Firestore, FCM, App Check)         │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 6   │ unzip -l meetway.apk | grep  │ Нашёл .so native библиотеки               │
│     │ lib/                         │                                           │
├─────┼──────────────────────────────┼───────────────────────────────────────────┤
│ 7   │ MobSF отчёт                  │ Сверил со своим анализом дополнительно    │
└─────┴─────────-----------------------------------------------------------------
```