# MASTG-TEST-0201: Runtime Use of APIs to Access External Storage
https://mas.owasp.org/MASTG/tests/android/MASVS-STORAGE/MASTG-TEST-0201/

Этот тест проверяет, какие API для работы с файлами использует приложение во время работы. Вместо того чтобы просто смотреть на появившиеся файлы, ты перехватываешь системные вызовы и смотришь, когда, как и какие файлы приложение пытается создать, прочитать или записать. Это позволяет не только увидеть финальный результат, но и понять, какие именно данные и куда пытается сохранить приложение, даже если оно потом их удаляет

------

тест на приложении [[0_MeetWay]]

создам скрипт FRIDA который будет перехватывать методы работы с файлами
в это время я буду активно взаимодействовать с приложением!

------

нужно найти методы работы с файлами

```q
- `getExternalStorageDirectory()` — получение корневой директории внешнего хранилища.
    
- `getExternalStoragePublicDirectory()` — получение стандартных папок (например, `DCIM`, `Pictures`).
    
- `getExternalFilesDir()` — получение папки для файлов приложения.
    
- `FileOutputStream` — запись в файл.
    
- `open()` — общий системный вызов для открытия файлов (может дать много шума, но поймает всё).
    
- `MediaStore` API — для сохранения файлов через медиа-провайдер (например, фото в галерею).
```

------

фрида везде стоит уже

```q
кидаю shell телефона

adb shell
su

запускаю сервер фриды на телефоне

cd /data/local/tmp
chmod 755 frida-server-17.9.1-android-arm64
nohup ./frida-server-17.9.1-android-arm64 &


НА МАКЕ В НОВОМ ТЕМИНАЛЕ 

// кидаю порт
adb forward tcp:27042 tcp:27042

смотрю запущенные процессы

frida-ps -U

```

-------

скрипт для фида

```js
cat > /Users/evgeniy/Desktop/storage_hooks.js << 'EOF'
// storage_hooks.js - Universal External Storage Access Hook Script
Java.perform(function() {
    console.log("[*] === External Storage Access Hook Script Started ===");

    // 1. Перехват getExternalStorageDirectory()
    try {
        var Environment = Java.use("android.os.Environment");
        Environment.getExternalStorageDirectory.implementation = function() {
            var result = this.getExternalStorageDirectory();
            console.log("[+] Environment.getExternalStorageDirectory() called. Result: " + result);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return result;
        };
        console.log("[+] Hooked: Environment.getExternalStorageDirectory()");
    } catch(e) { console.log("[-] Error hooking Environment: " + e); }

    // 2. Перехват getExternalFilesDir()
    try {
        var Context = Java.use("android.content.Context");
        var ContextWrapper = Java.use("android.content.ContextWrapper");
        ContextWrapper.getExternalFilesDir.implementation = function(type) {
            var result = this.getExternalFilesDir(type);
            console.log("[+] Context.getExternalFilesDir() called with type: " + type + ". Result: " + result);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return result;
        };
        console.log("[+] Hooked: Context.getExternalFilesDir()");
    } catch(e) { console.log("[-] Error hooking Context: " + e); }

    // 3. Перехват FileOutputStream (запись в файл)
    try {
        var FileOutputStream = Java.use("java.io.FileOutputStream");
        FileOutputStream.$init.overload('java.io.File').implementation = function(file) {
            console.log("[+] FileOutputStream created for file: " + file.getPath());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.$init(file);
        };
        console.log("[+] Hooked: FileOutputStream.<init>()");
    } catch(e) { console.log("[-] Error hooking FileOutputStream: " + e); }

    // 4. Перехват FileWriter (запись текстовых данных)
    try {
        var FileWriter = Java.use("java.io.FileWriter");
        FileWriter.$init.overload('java.io.File').implementation = function(file) {
            console.log("[+] FileWriter created for file: " + file.getPath());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.$init(file);
        };
        console.log("[+] Hooked: FileWriter.<init>()");
    } catch(e) { console.log("[-] Error hooking FileWriter: " + e); }

    // 5. Перехват MediaStore (сохранение файлов в медиа-библиотеку)
    try {
        var MediaStore = Java.use("android.provider.MediaStore");
        // Это сложнее, т.к. методы статические. Для примера перехватим insert
        // На практике может потребоваться более детальный хук
        console.log("[*] MediaStore hooks are more complex and may require additional implementation.");
    } catch(e) { console.log("[-] Error hooking MediaStore: " + e); }

    // 6. Дополнительно: перехват File.exists() и File.mkdirs() для контекста
    try {
        var File = Java.use("java.io.File");
        File.exists.implementation = function() {
            var path = this.getPath();
            var result = this.exists();
            if (path.indexOf("/storage/") !== -1 || path.indexOf("/sdcard/") !== -1) {
                console.log("[+] File.exists() called for external storage path: " + path + " -> " + result);
            }
            return result;
        };
        console.log("[+] Hooked: File.exists() for external storage paths");

        File.mkdirs.implementation = function() {
            var path = this.getPath();
            var result = this.mkdirs();
            if (path.indexOf("/storage/") !== -1 || path.indexOf("/sdcard/") !== -1) {
                console.log("[+] File.mkdirs() called for external storage path: " + path + " -> " + result);
            }
            return result;
        };
        console.log("[+] Hooked: File.mkdirs() for external storage paths");
    } catch(e) { console.log("[-] Error hooking File: " + e); }

    console.log("[*] === All hooks are set. Monitoring external storage access... ===");
});
EOF
```


 приложение со скриптом через frida

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/storage_hooks.js
```





РЕЗУЛЬТАТЫ РАБОТЫ СКРИПТА НИЖЕ

а сейчас краткий итог:

 В логах нет ни одного обращения к `/sdcard/` или `/storage/emulated/0/` - это значит, что приложение не пишет во внешнее хранилище, что хорошо
   
есть запись::

   ==/data/user/0/com.evgeniy.meetway/shared_prefs/meetway_secure_prefs.xml==
   
   Это файл предпочтений (SharedPreferences), который лежит во внутреннем хранилище и, скорее всего, содержит какие-то настройки безопасности, может быть, даже токены
   
множество записей в папку `cache/` - это видео и аудиофайлы, которые скачиваются для просмотра. Это обычное поведение для мессенджеров и соцсетей - кэшировать контент во внуреннем кеше
   

#### файл  `meetway_secure_prefs.xml`

меня интересует, не происходит ли запись во внешнее хранилище (`/sdcard/` или `/storage/emulated/0/`). Судя по логам, таких записей нет. Поэтому тест можно считать пройденным

но если файл `meetway_secure_prefs.xml` содержит токены или другие чувствительные данные в открытом виде, это может быть проблемой, но для этого есть отдельный тест на безопасность внутреннего хранилища



### разбор файла `meetway_secure_prefs.xml`

Читаю файл через root:

```sh
adb shell su -c 'cat /data/user/0/com.evgeniy.meetway/shared_prefs/meetway_secure_prefs.xml'
```

Результат:
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="__androidx_security_crypto_encrypted_prefs_key_keyset__">...12a7011465d72fc54e4ccb04f7ddcb8baec696b62b...</string>
    <string name="AQy0fkndIcXVDWTIurTzCZfdDE8NM59A2v4jxM0=">ATiDDlBSwk8NbmRoAKh3XQ80mMkYVmgNyLDj5X2tNZlghohjV59xfpI2B2CdJ+pKChL1</string>
    <string name="AQy0fkk1GVVPjy3XlFr7vXS07xrSYnB82EEzTda8sq4=">ATiDDlAfZ/o/Dca7l/z99pPiMO4DXu2dZqbkg9ov+A9HAcCfh7js98TseAS0cmtz+jqIjajbVVQ8jlY6Ta75bVu6R02h4DC+4e4bt+brng4KfuXyf6+2ZlEEQ/ijc7Abj/5rv5KB1OD6sqyuIOajrfV2CLWw+H2NyL/Zth3tHco7iL6pVhGfwMDxK+zU8ibDQHnw52g3oKerpmj0bPwYDeWTmjxaIzA9nZJg1Gc5rOCFU41YzXPkuSXypyxozebdHW5G5uhyc/jQoUmbd+hAGgNCeQDZx/TSnkKJ0k0/lHN/eqF1cMJJQmBXUa8X9QJm3R3hGhh/+9aJoJLsR6Tc9E4QzYmTzl3ywp8kMC91+sM1VPIr52GOMXg+cI0YCWo48XE3jQ57ebmc02NRwsc66X5iJf+XznYUtCiis/QzM11rzp09ChA=</string>
    <string name="__androidx_security_crypto_encrypted_prefs_value_keyset__">...128801ea4ef11dd515abb3a2f2e91796c5275b6270650a7e3252dceaac73522350bb3d4ffce8ffad1104962f9bee9617c2568bfcb8b64968107f7e5ccaa059a93f...</string>
</map>
```

### что это?

это **EncryptedSharedPreferences** (Google Tink + AES-GCM).

- `__androidx_security_crypto...keyset__` - ключи шифрования (Tink keyset)
- Все значения с префиксом `AQy0fk...` - это **зашифрованные данные** (ключи = AES-GCM encrypted, значения = тоже)
- Ничего в открытом виде нет

Файл использует `androidx.security.crypto.EncryptedSharedPreferences` - это аналог iOS Keychain на Android. Даже имея root-доступ к файлу, данные прочитать невозможно без ключа из Android KeyStore

**✅ Чувствительные данные (JWT, токены) хранятся в зашифрованном виде**

-------

### итого!

**✅ ТЕСТ ПРОЙДЕН**

Приложение MeetWay **не использует API для доступа к внешнему хранилищу**:

- ❌ Ни одного вызова `getExternalStorageDirectory()`
- ❌ Ни одного вызова `getExternalFilesDir()`
- ❌ Ни одной записи на `/sdcard/` или `/storage/emulated/0/`
- ✅ Все записи — только во внутреннее хранилище (`/data/user/0/com.evgeniy.meetway/cache/`, `shared_prefs/`)
- ✅ `SecureStorage` использует `EncryptedSharedPreferences` — данные зашифрованы
- ✅ Медиафайлы (аудио, видео) кэшируются во внутреннем `cache/` — недоступны другим приложениям

**Рисков утечки данных через внешнее хранилище нет**


Так что ничего страшного в этом файле нету, потому что он всё равно хрони зашифрованном виде, и достать его можно через другие динамические методы момент записи например. А это уже другой тест.





```c
evgeniy@Evgeniys-MacBook-Pro-2 ~ % frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/storage_hooks.js
     ____
    / _  |   Frida 17.9.1 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to KB2003 (id=6b62e119)
Spawned `com.evgeniy.meetway`. Resuming main thread!
[KB2003::com.evgeniy.meetway ]-> [*] === External Storage Access Hook Script Started ===
[+] Hooked: Environment.getExternalStorageDirectory()
[+] Hooked: Context.getExternalFilesDir()
[+] Hooked: FileOutputStream.<init>()
[+] Hooked: FileWriter.<init>()
[*] MediaStore hooks are more complex and may require additional implementation.
[+] Hooked: File.exists() for external storage paths
[+] Hooked: File.mkdirs() for external storage paths
[*] === All hooks are set. Monitoring external storage access... ===
[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/shared_prefs/meetway_secure_prefs.xml
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at android.app.SharedPreferencesImpl.createFileOutputStream(SharedPreferencesImpl.java:702)
	at android.app.SharedPreferencesImpl.writeToFile(SharedPreferencesImpl.java:793)
	at android.app.SharedPreferencesImpl.-$$Nest$mwriteToFile(SharedPreferencesImpl.java:0)
	at android.app.SharedPreferencesImpl$2.run(SharedPreferencesImpl.java:672)
	at android.app.QueuedWork.processPendingWork(QueuedWork.java:265)
	at android.app.QueuedWork.-$$Nest$smprocessPendingWork(QueuedWork.java:0)
	at android.app.QueuedWork$QueuedWorkHandler.handleMessage(QueuedWork.java:285)
	at android.os.Handler.dispatchMessage(Handler.java:106)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.os.HandlerThread.run(HandlerThread.java:67)

[KB2003::com.evgeniy.meetway ]-> [+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/files/profileInstalled
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at androidx.profileinstaller.ProfileVerifier$Cache.writeOnFile(ProfileVerifier.java:387)
	at androidx.profileinstaller.ProfileVerifier.writeProfileVerification(ProfileVerifier.java:282)
        at androidx.profileinstaller.ProfileInstaller.writeProfile(ProfileInstaller.java:568)
	at androidx.profileinstaller.ProfileInstaller.writeProfile(ProfileInstaller.java:506)
	at androidx.profileinstaller.ProfileInstaller.writeProfile(ProfileInstaller.java:470)
	at androidx.profileinstaller.ProfileInstallerInitializer.lambda$writeInBackground$2(ProfileInstallerInitializer.java:136)
	at androidx.profileinstaller.ProfileInstallerInitializer$$ExternalSyntheticLambda2.run(D8$$SyntheticClass:0)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1100)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:1572)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/audio_326F7016-62AA-4DBA-A2CA-E18CC9703D9B608024466452831492.m4a
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invokeSuspend(ObjectStorageService.kt:323)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invokeSuspend(ObjectStorageService.kt:305)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadAudio-0E7RQCE(ObjectStorageService.kt:304)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$AudioPlaybackContent$1$1$result$1.invokeSuspend(ChatsScreen.kt:918)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_40F406AF-CA38-45CD-9180-FA102AF844CD3422788223158023380.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_40F406AF-CA38-45CD-9180-FA102AF844CD3346554466859477903.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/audio_326F7016-62AA-4DBA-A2CA-E18CC9703D9B3813326909841914861.m4a
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invokeSuspend(ObjectStorageService.kt:323)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invokeSuspend(ObjectStorageService.kt:305)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadAudio-0E7RQCE(ObjectStorageService.kt:304)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$AudioPlaybackContent$1$1$result$1.invokeSuspend(ChatsScreen.kt:918)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/audio_72227D6E-8A6C-4C2D-8641-BFFFC57704554895398898904139306.m4a
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invokeSuspend(ObjectStorageService.kt:323)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invokeSuspend(ObjectStorageService.kt:305)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadAudio-0E7RQCE(ObjectStorageService.kt:304)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$AudioPlaybackContent$1$1$result$1.invokeSuspend(ChatsScreen.kt:918)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_257B5A61-9382-4499-961B-EF39FAC0D4834950006720775133387.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_257B5A61-9382-4499-961B-EF39FAC0D4839217551752663618879.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/audio_72227D6E-8A6C-4C2D-8641-BFFFC57704555002915934836148929.m4a
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invokeSuspend(ObjectStorageService.kt:323)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invokeSuspend(ObjectStorageService.kt:305)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadAudio-0E7RQCE(ObjectStorageService.kt:304)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$AudioPlaybackContent$1$1$result$1.invokeSuspend(ChatsScreen.kt:918)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/audio_A01BCE8B-9F4F-4BEA-8ABC-1CBDFC1863CE1043386950750956987.m4a
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invokeSuspend(ObjectStorageService.kt:323)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invokeSuspend(ObjectStorageService.kt:305)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadAudio$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadAudio-0E7RQCE(ObjectStorageService.kt:304)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$AudioPlaybackContent$1$1$result$1.invokeSuspend(ChatsScreen.kt:918)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9876DDDD-AEF6-4A13-BAD7-F0918458AF0C4124891464819674978.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9876DDDD-AEF6-4A13-BAD7-F0918458AF0C2889402319802834079.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9B168EAE-0470-4F92-952F-270D72435F83978587485087864865.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9B168EAE-0470-4F92-952F-270D72435F836357279229302309564.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9B168EAE-0470-4F92-952F-270D72435F832153611089045893423.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_9B168EAE-0470-4F92-952F-270D72435F835129014642182107997.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_6498229B-0305-4777-9182-07FF892B30023216128593912942503.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_6498229B-0305-4777-9182-07FF892B30021037081384585050907.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_4C564C28-D191-4723-97F9-F80CF4A9630F4844157165911154419.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_4C564C28-D191-4723-97F9-F80CF4A9630F3637228310581141758.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_43DF8659-CCA2-4C16-95DE-78FB39A9A5983792405560510776347.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_43DF8659-CCA2-4C16-95DE-78FB39A9A5981485546088382672777.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_292223D4-239B-40C9-84E9-4CE8CB94D2A31470052870819423346.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_292223D4-239B-40C9-84E9-4CE8CB94D2A36931462199895517068.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_E8682C3F-1D09-49E2-BA42-30FEC48B0F503895350010288241017.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_E8682C3F-1D09-49E2-BA42-30FEC48B0F50237053680024907762.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_8D502267-EA56-49BA-858A-BB0ACE89C6195677813336254780663.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$VideoCircleMessage$1$1$result$1.invokeSuspend(ChatsScreen.kt:372)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

[+] FileOutputStream created for file: /data/user/0/com.evgeniy.meetway/cache/video_8D502267-EA56-49BA-858A-BB0ACE89C6194687220228953348981.mp4
[+] Backtrace:
java.lang.Exception
	at java.io.FileOutputStream.<init>(Native Method)
	at kotlin.io.FilesKt__FileReadWriteKt.writeBytes(FileReadWrite.kt:114)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invokeSuspend(ObjectStorageService.kt:468)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2$1.invoke(ObjectStorageService.kt:2)
	at com.evgeniy.meetway.service.ObjectStorageService.retry-0E7RQCE(ObjectStorageService.kt:823)
	at com.evgeniy.meetway.service.ObjectStorageService.access$retry-0E7RQCE(ObjectStorageService.kt:42)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invokeSuspend(ObjectStorageService.kt:450)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:8)
	at com.evgeniy.meetway.service.ObjectStorageService$downloadVideo$2.invoke(ObjectStorageService.kt:4)
	at kotlinx.coroutines.intrinsics.UndispatchedKt.startUndispatchedOrReturn(Undispatched.kt:42)
	at kotlinx.coroutines.BuildersKt__Builders_commonKt.withContext(Builders.common.kt:164)
	at kotlinx.coroutines.BuildersKt.withContext(dex-id-d0e00a805465aee9510cf02fcdedc3ecec39dc39:1)
	at com.evgeniy.meetway.service.ObjectStorageService.downloadVideo-0E7RQCE(ObjectStorageService.kt:449)
	at com.evgeniy.meetway.ui.screens.chat.ChatsScreenKt$MessageBubbleComposable$internalVideoThumbnail$2$1$file$1.invokeSuspend(ChatsScreen.kt:559)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:113)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:589)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:823)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:720)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:707)

```



