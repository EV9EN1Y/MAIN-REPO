MASTG-TEST-0254: Dangerous App Permissions

MASVS категория: MASVS-PRIVACY
MASWE: MASWE-0117 (Inadequate Permission Management)

### О чём тест

Проверяет, какие опасные (dangerous) разрешения запрашивает приложение в
AndroidManifest.xml, и обоснованы ли они

Dangerous permissions - это те, что требуют runtime-согласия пользователя: камера,
микрофон, геолокация, контакты, SMS, телефон, storage и т.д. Они перечислены в
Android developer docs

### Зачем нужен

Приложение не должно запрашивать разрешения, которые не нужны для его основной
функциональности. Если MeetWay (социальная сеть с чатами) запрашивает READ_SMS или
RECORD_AUDIO без объективной причины – это нарушение. Пользователь должен
понимать, зачем приложению доступ к его данным-

### Как выполнить

Всё просто, ничего настраивать не нужно:

1. Декомпилировать APK (или взять исходники):

```q
 apktool d meetway.apk
```
   Или просто открыть AndroidManifest.xml в проекте
   
2. Найти все `<uses-permission>` в AndroidManifest.xml
3. Сравнить со списком dangerous permissions. Dangerous на Android - это группы:
    - CAMERA - доступ к камере
    - RECORD_AUDIO - доступ к микрофону
    - ACCESS_FINE_LOCATION / ACCESS_COARSE_LOCATION - геолокация
    - READ_CONTACTS / WRITE_CONTACTS - контакты
    - READ_EXTERNAL_STORAGE / WRITE_EXTERNAL_STORAGE - файлы
    - READ_SMS / SEND_SMS / RECEIVE_SMS - SMS
    - READ_PHONE_STATE / CALL_PHONE - телефон
    - RECORD_AUDIO - аудио
    - BODY_SENSORS - датчики
    - ACTIVITY_RECOGNITION - активность
    - POST_NOTIFICATIONS - уведомления (Android 13+)
    - NEARBY_DEVICES / BLUETOOTH_SCAN / BLUETOOTH_CONNECT - BLE/WiFi (Android 12+)
    - READ_MEDIA_IMAGES / READ_MEDIA_VIDEO / READ_MEDIA_AUDIO - медиа (Android
      13+)
4. Вывод: если приложение декларирует опасные разрешения, которых нет в его
   функционале - тест не пройден


------

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

открыл манифест

```xml
<?xml version="1.0" encoding="utf-8"?>  
<manifest xmlns:android="http://schemas.android.com/apk/res/android"  
    android:versionCode="1"  
    android:versionName="1.0"  
    android:compileSdkVersion="36"  
    android:compileSdkVersionCodename="16"  
    package="com.evgeniy.meetway"  
    platformBuildVersionCode="36"  
    platformBuildVersionName="16">  
    <uses-sdk  
        android:minSdkVersion="33"  
        android:targetSdkVersion="36"/>  
    <uses-permission android:name="android.permission.INTERNET"/>  
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>  
    <uses-permission android:name="android.permission.CAMERA"/>  
    <uses-feature  
        android:name="android.hardware.camera"  
        android:required="false"/>  
    <uses-feature  
        android:name="android.hardware.camera.autofocus"  
        android:required="false"/>  
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>  
    <uses-permission android:name="android.permission.VIBRATE"/>  
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>  
    <uses-permission  
        android:name="android.permission.READ_EXTERNAL_STORAGE"  
        android:maxSdkVersion="32"/>  
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>  
    <uses-permission android:name="android.permission.WAKE_LOCK"/>  
    <uses-permission android:name="android.permission.USE_BIOMETRIC"/>  
    <uses-permission android:name="android.permission.USE_FINGERPRINT"/>  
    <uses-permission android:name="com.google.android.c2dm.permission.RECEIVE"/>  
    <uses-permission android:name="com.google.android.providers.gsf.permission.READ_GSERVICES"/>  
    <permission  
        android:name="com.evgeniy.meetway.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"  
        android:protectionLevel="signature"/>  
    <uses-permission android:name="com.evgeniy.meetway.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"/>  
    <application  
        android:theme="@style/Theme.MeetWay"  
        android:label="@string/app_name"  
        android:icon="@mipmap/ic_launcher"  
        android:name="com.evgeniy.meetway.MeetWayApp"  
        android:debuggable="true"  
        android:testOnly="true"  
        android:allowBackup="false"  
        android:supportsRtl="true"  
        android:extractNativeLibs="false"  
        android:fullBackupContent="@xml/backup_rules"  
        android:networkSecurityConfig="@xml/network_security_config"  
        android:roundIcon="@mipmap/ic_launcher_round"  
        android:appComponentFactory="androidx.core.app.CoreComponentFactory"  
        android:dataExtractionRules="@xml/data_extraction_rules">  
        <activity  
            android:theme="@style/Theme.MeetWay"  
            android:name="com.evgeniy.meetway.AuthRedirectActivity"  
            android:exported="true">  
            <intent-filter>  
                <action android:name="android.intent.action.VIEW"/>  
                <category android:name="android.intent.category.DEFAULT"/>  
                <category android:name="android.intent.category.BROWSABLE"/>  
                <data  
                    android:scheme="yx547697fe6e9d46ea9a7e538922d1425d"  
                    android:host="auth"/>  
            </intent-filter>  
        </activity>  
        <activity  
            android:theme="@style/Theme.MeetWay"  
            android:name="com.evgeniy.meetway.YandexAuthWebViewActivity"  
            android:exported="false"/>  
        <activity  
            android:theme="@style/Theme.MeetWay"  
            android:label="@string/app_name"  
            android:name="com.evgeniy.meetway.MainActivity"  
            android:exported="true"  
            android:windowSoftInputMode="adjustResize">  
            <intent-filter>  
                <action android:name="android.intent.action.MAIN"/>  
                <category android:name="android.intent.category.LAUNCHER"/>  
            </intent-filter>  
        </activity>  
        <service  
            android:name="com.evgeniy.meetway.service.MeetWayMessagingService"  
            android:exported="false">  
            <intent-filter>  
                <action android:name="com.google.firebase.MESSAGING_EVENT"/>  
            </intent-filter>  
        </service>  
        <receiver  
            android:name="com.evgeniy.meetway.service.MeetWayMessagingService.NotificationActionReceiver"  
            android:exported="false">  
            <intent-filter>  
                <action android:name="REPLY_ACTION"/>  
                <action android:name="MARK_AS_READ"/>  
            </intent-filter>  
        </receiver>  
        <service  
            android:name="androidx.camera.core.impl.MetadataHolderService"  
            android:enabled="false"  
            android:exported="false">  
            <meta-data  
                android:name="androidx.camera.core.impl.MetadataHolderService.DEFAULT_CONFIG_PROVIDER"  
                android:value="androidx.camera.camera2.Camera2Config$DefaultProvider"/>  
        </service>  
        <activity  
            android:theme="@android:style/Theme.Translucent.NoTitleBar"  
            android:name="com.google.firebase.auth.internal.GenericIdpActivity"  
            android:exported="true"  
            android:excludeFromRecents="true"  
            android:launchMode="singleTask">  
            <intent-filter>  
                <action android:name="android.intent.action.VIEW"/>  
                <category android:name="android.intent.category.DEFAULT"/>  
                <category android:name="android.intent.category.BROWSABLE"/>  
                <data  
                    android:scheme="genericidp"  
                    android:host="firebase.auth"  
                    android:path="/"/>  
            </intent-filter>  
        </activity>  
        <activity  
            android:theme="@android:style/Theme.Translucent.NoTitleBar"  
            android:name="com.google.firebase.auth.internal.RecaptchaActivity"  
            android:exported="true"  
            android:excludeFromRecents="true"  
            android:launchMode="singleTask">  
            <intent-filter>  
                <action android:name="android.intent.action.VIEW"/>  
                <category android:name="android.intent.category.DEFAULT"/>  
                <category android:name="android.intent.category.BROWSABLE"/>  
                <data  
                    android:scheme="recaptcha"  
                    android:host="firebase.auth"  
                    android:path="/"/>  
            </intent-filter>  
        </activity>  
        <service  
            android:name="com.google.firebase.components.ComponentDiscoveryService"  
            android:exported="false"  
            android:directBootAware="true">  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.auth.FirebaseAuthRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.firestore.FirebaseFirestoreKtxRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.firestore.FirestoreRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.messaging.FirebaseMessagingKtxRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.messaging.FirebaseMessagingRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.installations.FirebaseInstallationsKtxRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.installations.FirebaseInstallationsRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.FirebaseCommonKtxRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
            <meta-data  
                android:name="com.google.firebase.components:com.google.firebase.datatransport.TransportRegistrar"  
                android:value="com.google.firebase.components.ComponentRegistrar"/>  
        </service>  
        <activity  
            android:theme="@android:style/Theme.Material.Light.NoActionBar"  
            android:name="androidx.activity.ComponentActivity"  
            android:exported="true"/>  
        <activity  
            android:name="androidx.compose.ui.tooling.PreviewActivity"  
            android:exported="true"/>  
        <receiver  
            android:name="com.google.firebase.iid.FirebaseInstanceIdReceiver"  
            android:permission="com.google.android.c2dm.permission.SEND"  
            android:exported="true">  
            <intent-filter>  
                <action android:name="com.google.android.c2dm.intent.RECEIVE"/>  
            </intent-filter>  
            <meta-data  
                android:name="com.google.android.gms.cloudmessaging.FINISHED_AFTER_HANDLED"  
                android:value="true"/>  
        </receiver>  
        <service  
            android:name="com.google.firebase.messaging.FirebaseMessagingService"  
            android:exported="false"  
            android:directBootAware="true">  
            <intent-filter android:priority="-500">  
                <action android:name="com.google.firebase.MESSAGING_EVENT"/>  
            </intent-filter>  
        </service>  
        <service  
            android:name="androidx.credentials.playservices.CredentialProviderMetadataHolder"  
            android:enabled="true"  
            android:exported="false">  
            <meta-data  
                android:name="androidx.credentials.CREDENTIAL_PROVIDER_KEY"  
                android:value="androidx.credentials.playservices.CredentialProviderPlayServicesImpl"/>  
        </service>  
        <activity  
            android:theme="@style/Theme.Hidden"  
            android:name="androidx.credentials.playservices.HiddenActivity"  
            android:enabled="true"  
            android:exported="false"  
            android:configChanges="screenSize|screenLayout|orientation|keyboardHidden"  
            android:fitsSystemWindows="true"/>  
        <activity  
            android:theme="@android:style/Theme.Translucent.NoTitleBar"  
            android:name="com.google.android.gms.auth.api.signin.internal.SignInHubActivity"  
            android:exported="false"  
            android:excludeFromRecents="true"/>  
        <service  
            android:name="com.google.android.gms.auth.api.signin.RevocationBoundService"  
            android:permission="com.google.android.gms.auth.api.signin.permission.REVOCATION_NOTIFICATION"  
            android:exported="true"  
            android:visibleToInstantApps="true"/>  
        <provider  
            android:name="com.google.firebase.provider.FirebaseInitProvider"  
            android:exported="false"  
            android:authorities="com.evgeniy.meetway.firebaseinitprovider"  
            android:initOrder="100"  
            android:directBootAware="true"/>  
        <activity  
            android:theme="@android:style/Theme.Translucent.NoTitleBar"  
            android:name="com.google.android.gms.common.api.GoogleApiActivity"  
            android:exported="false"/>  
        <meta-data  
            android:name="com.google.android.gms.version"  
            android:value="@integer/google_play_services_version"/>  
        <provider  
            android:name="androidx.startup.InitializationProvider"  
            android:exported="false"  
            android:authorities="com.evgeniy.meetway.androidx-startup">  
            <meta-data  
                android:name="androidx.emoji2.text.EmojiCompatInitializer"  
                android:value="androidx.startup"/>  
            <meta-data  
                android:name="androidx.lifecycle.ProcessLifecycleInitializer"  
                android:value="androidx.startup"/>  
            <meta-data  
                android:name="androidx.profileinstaller.ProfileInstallerInitializer"  
                android:value="androidx.startup"/>  
        </provider>  
        <receiver  
            android:name="androidx.profileinstaller.ProfileInstallReceiver"  
            android:permission="android.permission.DUMP"  
            android:enabled="true"  
            android:exported="true"  
            android:directBootAware="false">  
            <intent-filter>  
                <action android:name="androidx.profileinstaller.action.INSTALL_PROFILE"/>  
            </intent-filter>  
            <intent-filter>  
                <action android:name="androidx.profileinstaller.action.SKIP_FILE"/>  
            </intent-filter>  
            <intent-filter>  
                <action android:name="androidx.profileinstaller.action.SAVE_PROFILE"/>  
            </intent-filter>  
            <intent-filter>  
                <action android:name="androidx.profileinstaller.action.BENCHMARK_OPERATION"/>  
            </intent-filter>  
        </receiver>  
        <service  
            android:name="com.google.android.datatransport.runtime.backends.TransportBackendDiscovery"  
            android:exported="false">  
            <meta-data  
                android:name="backend:com.google.android.datatransport.cct.CctBackendFactory"  
                android:value="cct"/>  
        </service>  
        <service  
            android:name="com.google.android.datatransport.runtime.scheduling.jobscheduling.JobInfoSchedulerService"  
            android:permission="android.permission.BIND_JOB_SERVICE"  
            android:exported="false"/>  
        <receiver  
            android:name="com.google.android.datatransport.runtime.scheduling.jobscheduling.AlarmManagerSchedulerBroadcastReceiver"  
            android:exported="false"/>  
        <activity  
            android:theme="@style/Theme.PlayCore.Transparent"  
            android:name="com.google.android.play.core.common.PlayCoreDialogWrapperActivity"  
            android:exported="false"  
            android:stateNotNeeded="true"/>  
    </application>  
</manifest>
```


---------

```c
ВОТ РАзрешения

┌─────────────────────────────────────────────┬───────────┬──────────────────────┐
│ Разрешение                                  │ Тип       │ Назначение           │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ INTERNET                                    │ Normal    │ Доступ в интернет    │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ ACCESS_NETWORK_STATE                        │ Normal    │ Состояние сети       │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ CAMERA                                      │ Dangerous │ Фото профиля         │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ RECORD_AUDIO                                │ Dangerous │ Голосовые сообщения  │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ VIBRATE                                     │ Normal    │ Вибрация             │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ POST_NOTIFICATIONS                          │ Dangerous │ Push-уведомления     │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ READ_EXTERNAL_STORAGE                       │ Dangerous │ Фото (до API 32)     │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ READ_MEDIA_IMAGES                           │ Dangerous │ Фото (API 33+)       │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ WAKE_LOCK                                   │ Normal    │ Не даёт экрану       │
│                                             │           │ гаснуть              │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ USE_BIOMETRIC                               │ Normal    │ Биометрия            │
│                                             │           │ (FaceID/TouchID)     │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ USE_FINGERPRINT                             │ Normal    │ Отпечаток пальца     │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ com.google.android.c2dm.permission.RECEIVE  │ Signature │ Firebase Cloud       │
│                                             │           │ Messaging            │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ com.google.android.providers.gsf.permission │ Signature │ Google Services      │
│ .READ_GSERVICES                             │           │                      │
├─────────────────────────────────────────────┼───────────┼──────────────────────┤
│ com.evgeniy.meetway.DYNAMIC_RECEIVER_NOT_EX │ Signature │ Внутреннее системное │
│ PORTED_PERMISSION                           │           │                      │
└─────────────────────────────────────────────┴───────────┴──────────────────────┘

```

я просмотрел функционал приложения и могу сказать, что все данные разрешения обоснованны и без них приложение не сможет выполнять свои функции, "левых" разрешений нет, для каждого разрешения - есть соответствующий и понятный функционал в приложении, который понятен

## тест пройден