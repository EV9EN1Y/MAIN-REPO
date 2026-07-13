# MASTG-TEST-0262: References to Backup Configurations Not Excluding Sensitive Data

https://mas.owasp.org/MASTG/tests/android/MASVS-STORAGE/MASTG-TEST-0262/

-------

Тест проверяет, правильно ли настроено приложение для исключения чувствительных данных из резервных копий. Он анализирует файл 
`AndroidManifest.xml` 
и правила бэкапа (файлы 
`backup_rules.xml` или `data_extraction_rules.xml`)


- `   android:allowBackup` - если этот флаг установлен в `true` (или отсутствует, что по умолчанию `true`), то бэкап разрешён.
    
- `android:fullBackupContent` - должен указывать на XML-файл с правилами для Android 11 и ниже.
    
- `android:dataExtractionRules` - должен указывать на XML-файл с правилами для Android 12 и выше.

------

**Тест ПРОВАЛЕН**, если одновременно выполняются условия:

1. `android:allowBackup="true"` (или отсутствует, что эквивалентно `true`).
    
2. Отсутствуют правила `fullBackupContent` или `dataExtractionRules`, или они не исключают все файлы с чувствительными данными.
    

**Тест ПРОЙДЕН**, если:

- `android:allowBackup="false"` (бэкап отключён полностью).
    
- ИЛИ есть правила, которые явно исключают все файлы, содержащие чувствительные данные.

------



ВОТ МАНИФЕСТ

```xml
**<?xml version=**"1.0" **encoding=**"utf-8"**?>**

**<manifest** xmlns**:**android**=**"http://schemas.android.com/apk/res/android"

    xmlns**:**tools**=**"http://schemas.android.com/tools"**>**

  

    <!-- Permissions -->

    **<uses-permission** android**:**name**=**"android.permission.INTERNET" **/>**

    **<uses-permission** android**:**name**=**"android.permission.ACCESS_NETWORK_STATE" **/>**

    **<uses-permission** android**:**name**=**"android.permission.CAMERA" **/>**

    **<uses-feature** android**:**name**=**"android.hardware.camera" android**:**required**=**"false" **/>**

    **<uses-feature** android**:**name**=**"android.hardware.camera.autofocus" android**:**required**=**"false" **/>**

    **<uses-permission** android**:**name**=**"android.permission.RECORD_AUDIO" **/>**

    **<uses-permission** android**:**name**=**"android.permission.VIBRATE" **/>**

    **<uses-permission** android**:**name**=**"android.permission.POST_NOTIFICATIONS" **/>**

    **<uses-permission** android**:**name**=**"android.permission.READ_EXTERNAL_STORAGE"

        android**:**maxSdkVersion**=**"32" **/>**

    **<uses-permission** android**:**name**=**"android.permission.READ_MEDIA_IMAGES" **/>**

  

    **<application**

        android**:**name**=**".MeetWayApp"

✅👉✅👉✅👉
✅👉✅👉✅👉
✅👉✅👉✅👉 БЕКАП ВЫКЛЮЧЕН _ ОТЛИЧНО_✅👉     android**:**allowBackup**=**"false"  
✅👉✅👉✅👉
✅👉✅👉✅👉


        android**:**dataExtractionRules**=**"@xml/data_extraction_rules"

        android**:**fullBackupContent**=**"@xml/backup_rules"

        android**:**icon**=**"@mipmap/ic_launcher"

        android**:**label**=**"@string/app_name"

        android**:**roundIcon**=**"@mipmap/ic_launcher_round"

        android**:**supportsRtl**=**"true"

        android**:**theme**=**"@style/Theme.MeetWay"

        android**:**networkSecurityConfig**=**"@xml/network_security_config"

        tools**:**targetApi**=**"36"**>**

        <!-- Yandex OAuth Redirect (запасной вариант) -->

        **<activity**

            android**:**name**=**".AuthRedirectActivity"

            android**:**exported**=**"true"

            android**:**theme**=**"@style/Theme.MeetWay"**>**

            **<intent-filter>**

                **<action** android**:**name**=**"android.intent.action.VIEW" **/>**

                **<category** android**:**name**=**"android.intent.category.DEFAULT" **/>**

                **<category** android**:**name**=**"android.intent.category.BROWSABLE" **/>**

                <!-- Кастомная схема: yx547697fe6e9d46ea9a7e538922d1425d://auth -->

                **<data**

                    android**:**scheme**=**"yx547697fe6e9d46ea9a7e538922d1425d"

                    android**:**host**=**"auth" **/>**

            **</intent-filter>**

        **</activity>**

  

        <!-- Yandex OAuth WebView (основной вариант — перехват редиректа внутри приложения) -->

        **<activity**

            android**:**name**=**".YandexAuthWebViewActivity"

            android**:**exported**=**"false"

            android**:**theme**=**"@style/Theme.MeetWay" **/>**

  

        **<activity**

            android**:**name**=**".MainActivity"

            android**:**exported**=**"true"

            android**:**label**=**"@string/app_name"

            android**:**theme**=**"@style/Theme.MeetWay"

            android**:**windowSoftInputMode**=**"adjustResize"**>**

            **<intent-filter>**

                **<action** android**:**name**=**"android.intent.action.MAIN" **/>**

                **<category** android**:**name**=**"android.intent.category.LAUNCHER" **/>**

            **</intent-filter>**

        **</activity>**

  

        <!-- FCM Push Messaging Service -->

        **<service**

            android**:**name**=**".service.MeetWayMessagingService"

            android**:**exported**=**"false"**>**

            **<intent-filter>**

                **<action** android**:**name**=**"com.google.firebase.MESSAGING_EVENT" **/>**

            **</intent-filter>**

        **</service>**

  

        <!-- BroadcastReceiver для действий push-уведомлений (iOS: UNNotificationAction) -->

        **<receiver**

            android**:**name**=**".service.MeetWayMessagingService$NotificationActionReceiver"

            android**:**exported**=**"false"**>**

            **<intent-filter>**

                **<action** android**:**name**=**"REPLY_ACTION" **/>**

                **<action** android**:**name**=**"MARK_AS_READ" **/>**

            **</intent-filter>**

        **</receiver>**

    **</application>**

  

**</manifest>**
```



-----

А ВОТ ФАЙЛ С ПРАВИЛАМИ БЕКАПА 

Ну здесь пусто, потому что в принципе и сам быка отключён в манифесте, поэтому он создаваться не будет, поэтому тест считается пройдённым.

```xml
**<?xml version=**"1.0" **encoding=**"utf-8"**?>**<!--

   Sample backup rules file; uncomment and customize as necessary.

   See https://developer.android.com/guide/topics/data/autobackup

   for details.

   Note: This file is ignored for devices older than API 31

   See https://developer.android.com/about/versions/12/backup-restore

-->

**<full-backup-content>**

    <!--

   <include domain="sharedpref" path="."/>

   <exclude domain="sharedpref" path="device.xml"/>

-->

**</full-backup-content>**
```


-------

Тест пройдён успешно, так как бэкап для этого приложения и файлов в приложении отключён, это значит что если делать резервное копирование телефона, то никакие файлы с этого приложения не попадают в бэкап.

Файлы без правилами для бэкапа нужно в идеале указывать исключение для чувствительных файлов, как например в GitHub есть гиt игнор папка, которую добавляем файлы которые не будут отображаться для общего доступа, так же самое тут нужно в правилах указать какие файлы не должны попадать БК соответственно если БК разрешенных в манифесте тогда в папке с правилами должны быть у каждой файлы которые не нужны в него попадать.