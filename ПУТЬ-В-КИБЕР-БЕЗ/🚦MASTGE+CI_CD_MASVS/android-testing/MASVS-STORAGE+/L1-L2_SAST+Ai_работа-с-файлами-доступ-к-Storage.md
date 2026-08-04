# MASTG-TEST-0202: References to APIs and Permissions for Accessing External Storage
https://mas.owasp.org/MASTG/tests/android/MASVS-STORAGE/MASTG-TEST-0202/

------

 тест проверяет, есть ли в коде приложения упоминания API и разрешений, связанных с доступом к внешнему хранилищу (`External Storage`). Он не анализирует сами данные, а только смотрит, _пытается_ ли приложение получить доступ к общим папкам

**Ключевые моменты:**

- **API для доступа:**
- `getExternalStoragePublicDirectory()`, 
- `getExternalStorageDirectory()`,
- `getExternalFilesDir()`,
- `getExternalCacheDir()`,
- `MediaStore`
   
- **Разрешения в манифесте:** 
- `WRITE_EXTERNAL_STORAGE`, 
- `MANAGE_EXTERNAL_STORAGE`,
- `READ_EXTERNAL_STORAGE`.-----

нужно проверить  манифест (`AndroidManifest.xml`)

 APK в JADX. Найти файл `AndroidManifest.xml`  можно в корне дерева. Просмотреть его и найти, есть ли там разрешения:

- `<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />`
    
- `<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />`
    
- `<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" />"
    

поиск в Java/Kotlin коде:

- `getExternalStoragePublicDirectory`
    
- `getExternalStorageDirectory`
    
- `getExternalFilesDir`
    
- `getExternalCacheDir`
    
- `MediaStore`
    
- `WRITE_EXTERNAL_STORAGE` (ищем в строковых литералах или контексте)
    

Анализ найденных мест!

----------

тестирую приложение [[0_MeetWay]]

##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

открыл манифест и код, смотрю что нашел

вот тут он лежит

```
  android-meetway/app/src/main/AndroidManifest.xml
  
  
  
  
  
  evgeniy@Evgeniys-MacBook-Pro-2 ~ % cat /Users/evgeniy/Documents/android-meetway/app/src/main/AndroidManifest.xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="false" />
    <uses-feature android:name="android.hardware.camera.autofocus" android:required="false" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.VIBRATE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

    <application
        android:name=".MeetWayApp"
        android:allowBackup="false"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.MeetWay"
        android:networkSecurityConfig="@xml/network_security_config"
        tools:targetApi="36">
        <!-- Yandex OAuth Redirect (запасной вариант) -->
        <activity
            android:name=".AuthRedirectActivity"
            android:exported="true"
            android:theme="@style/Theme.MeetWay">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <!-- Кастомная схема: yx547697fe6e9d46ea9a7e538922d1425d://auth -->
                <data
                    android:scheme="yx547697fe6e9d46ea9a7e538922d1425d"
                    android:host="auth" />
            </intent-filter>
        </activity>

        <!-- Yandex OAuth WebView (основной вариант – перехват редиректа внутри приложения) -->
        <activity
            android:name=".YandexAuthWebViewActivity"
            android:exported="false"
            android:theme="@style/Theme.MeetWay" />

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.MeetWay"
            android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- FCM Push Messaging Service -->
        <service
            android:name=".service.MeetWayMessagingService"
            android:exported="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>

        <!-- BroadcastReceiver для действий push-уведомлений (iOS: UNNotificationAction) -->
        <receiver
            android:name=".service.MeetWayMessagingService$NotificationActionReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="REPLY_ACTION" />
                <action android:name="MARK_AS_READ" />
            </intent-filter>
        </receiver>
    </application>

</manifest>%
evgeniy@Evgeniys-MacBook-Pro-2 ~ %
```

----

### что в манифесте

открываю `AndroidManifest.xml` - смотрю что там по пермишенам к хранилищу

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
```

что есть:
- `READ_EXTERNAL_STORAGE` - только чтение, только до Android 13
- `READ_MEDIA_IMAGES` - для Android 13+, вместо старого READ_EXTERNAL_STORAGE

чего **нет**:
- `WRITE_EXTERNAL_STORAGE` ❌
- `MANAGE_EXTERNAL_STORAGE` ❌

уже хорошо - приложение не просит права на запись в общую папку

----

### смотрю код на API внешнего хранилища

проверил весь код - `getExternalStorageDirectory()`, `getExternalStoragePublicDirectory()`, `getExternalFilesDir()`, `getExternalCacheDir()` - **ни одного вызова**

но нашел одну штуку:

**MediaStore** - в ChatsScreen.kt есть функция `saveVideoToGallery()`

вот она

```kotlin
private fun saveVideoToGallery(context: Context, videoUrl: String, onResult: ...) {
    val contentValues = ContentValues().apply {
        put(MediaStore.Video.Media.DISPLAY_NAME, "video_${currentTimeMillis()}.mp4")
        put(MediaStore.Video.Media.MIME_TYPE, "video/mp4")
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            put(MediaStore.Video.Media.IS_PENDING, 1)
        }
    }
    val resolver = context.contentResolver
    val videoUri = resolver.insert(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, contentValues)
    if (videoUri != null) {
        resolver.openOutputStream(videoUri)?.use { outputStream ->
            resolver.openInputStream(uri)?.use { inputStream ->
                inputStream.copyTo(outputStream)
            }
        }
    }
}
```

это функция сохранения видео в галерею - пользователь может сохранить видео из чата к себе в альбом

на Android 10+ (API 29+) MediaStore insert работает вообще без WRITE_EXTERNAL_STORAGE, система сама дает доступ через content provider

но есть нюанс - на Android 9 и ниже для этой функции нужен `WRITE_EXTERNAL_STORAGE`, которого в манифесте нет. хотя с учётом что minSdk = 33, это вообще не актуально - код для старых версий просто не сработает, но и не крашнет... кароч - нормис

----

### итог

| что проверял                                      | результат                                                                            |
| ------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `WRITE_EXTERNAL_STORAGE`                          | ❌ нет - не просит                                                                    |
| `MANAGE_EXTERNAL_STORAGE`                         | ❌ нет - не просит                                                                    |
| `READ_EXTERNAL_STORAGE`                           | ✅ есть, только до SDK 32                                                             |
| `getExternalStorageDirectory()`                   | ❌ не используется                                                                    |
| `getExternalFilesDir()` / `getExternalCacheDir()` | ❌ не используется                                                                    |
| `MediaStore`                                      | ✅ есть - сохранение видео в галерею (через content provider, не через прямой доступ) |

**вывод:** приложение не использует прямые API для доступа к внешнему хранилищу. единственное место - `MediaStore` для сохранения видео в галерею, и это сделано правильно через content provider, без излишних прав

###  ВЫВОД = тест MASTG-TEST-0202: **ПРОЙДЕН** 

опасных API и пермишенов для внешнего хранилища нет
