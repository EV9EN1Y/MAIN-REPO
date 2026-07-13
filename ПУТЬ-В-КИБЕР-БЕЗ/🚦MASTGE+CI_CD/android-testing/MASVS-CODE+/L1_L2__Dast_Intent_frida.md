# MASTG-TEST-0375: Missing Validation of Data Returned from Implicit Intents
https://mas.owasp.org/MASTG/tests/android/MASVS-CODE/MASTG-TEST-0375/

**проверяет ли приложение данные, полученные в ответ на неявный интент**. Если приложение просто доверяет данным, которые вернуло другое приложение, злоумышленник может подсунуть поддельные данные и обмануть приложение

-------------------------

Приложение может отправлять неявный интент с запросом данных (например, выбрать файл, фото, контакт). Другое приложение обрабатывает этот запрос и возвращает результат. Если приложение не проверяет, что именно вернулось (URI, тип данных, содержимое), злоумышленник может создать приложение, которое вернет опасные данные и обманет твое приложение

тест - динамический.  нужно запустить приложение и перехватить, как оно обрабатывает ответы от неявных интентов

-------------

нужен скрипт фриды - который перехватит методы:

- `startActivityForResult` — отправка запроса.
    
- `onActivityResult` — получение результата.
    
- `getData` — чтение URI из результата.
    
- `getExtras` — чтение дополнительных данных.
    
- `getClipData` — чтение данных из буфера обмена.
    
- `ContentResolver.query` — чтение данных из провайдера.

---


```c
cat > /Users/evgeniy/Desktop/hook_intent_results.js << 'EOF'
// hook_intent_results.js - Перехват обработки результатов от неявных интентов
Java.perform(function() {
    console.log("[*] === Intent Result Hook Script Started ===");

    // 1. Перехват startActivityForResult
    try {
        var Activity = Java.use("android.app.Activity");
        Activity.startActivityForResult.overload('android.content.Intent', 'int').implementation = function(intent, requestCode) {
            console.log("[+] startActivityForResult() called");
            console.log("[+] Request code: " + requestCode);
            console.log("[+] Intent: " + intent);
            console.log("[+] Action: " + intent.getAction());
            console.log("[+] Data: " + intent.getData());
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.startActivityForResult(intent, requestCode);
        };
        console.log("[+] Hooked: Activity.startActivityForResult()");
    } catch(e) {
        console.log("[-] Error hooking startActivityForResult: " + e);
    }

    // 2. Перехват onActivityResult (через ActivityResultCallback)
    try {
        var ActivityResultCallback = Java.use("androidx.activity.result.ActivityResultCallback");
        ActivityResultCallback.onActivityResult.implementation = function(result) {
            console.log("[+] ActivityResultCallback.onActivityResult() called");
            console.log("[+] Result: " + result);
            console.log("[+] Result code: " + result.getResultCode());
            var data = result.getData();
            if (data) {
                console.log("[+] Data intent: " + data);
                console.log("[+] Data action: " + data.getAction());
                console.log("[+] Data URI: " + data.getData());
                console.log("[+] Extras: " + data.getExtras());
                if (data.getClipData()) {
                    console.log("[+] ClipData: " + data.getClipData());
                }
            }
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.onActivityResult(result);
        };
        console.log("[+] Hooked: ActivityResultCallback.onActivityResult()");
    } catch(e) {
        console.log("[-] Error hooking ActivityResultCallback: " + e);
    }

    // 3. Перехват Intent.getData()
    try {
        var Intent = Java.use("android.content.Intent");
        Intent.getData.implementation = function() {
            var result = this.getData();
            console.log("[+] Intent.getData() called -> " + result);
            return result;
        };
        console.log("[+] Hooked: Intent.getData()");
    } catch(e) {
        console.log("[-] Error hooking Intent.getData: " + e);
    }

    // 4. Перехват Intent.getExtras()
    try {
        var Intent = Java.use("android.content.Intent");
        Intent.getExtras.implementation = function() {
            var result = this.getExtras();
            console.log("[+] Intent.getExtras() called -> " + result);
            return result;
        };
        console.log("[+] Hooked: Intent.getExtras()");
    } catch(e) {
        console.log("[-] Error hooking Intent.getExtras: " + e);
    }

    // 5. Перехват Intent.getClipData()
    try {
        var Intent = Java.use("android.content.Intent");
        Intent.getClipData.implementation = function() {
            var result = this.getClipData();
            console.log("[+] Intent.getClipData() called -> " + result);
            return result;
        };
        console.log("[+] Hooked: Intent.getClipData()");
    } catch(e) {
        console.log("[-] Error hooking Intent.getClipData: " + e);
    }

    // 6. Перехват ContentResolver.query() для проверки данных из провайдера
    try {
        var ContentResolver = Java.use("android.content.ContentResolver");
        ContentResolver.query.overload('android.net.Uri', '[Ljava.lang.String;', 'java.lang.String', '[Ljava.lang.String;', 'java.lang.String').implementation = function(uri, projection, selection, selectionArgs, sortOrder) {
            console.log("[+] ContentResolver.query() called");
            console.log("[+] URI: " + uri);
            console.log("[+] Projection: " + projection);
            console.log("[+] Selection: " + selection);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.query(uri, projection, selection, selectionArgs, sortOrder);
        };
        console.log("[+] Hooked: ContentResolver.query()");
    } catch(e) {
        console.log("[-] Error hooking ContentResolver.query: " + e);
    }

    // 7. Перехват ContentResolver.openInputStream()
    try {
        var ContentResolver = Java.use("android.content.ContentResolver");
        ContentResolver.openInputStream.implementation = function(uri) {
            console.log("[+] ContentResolver.openInputStream() called");
            console.log("[+] URI: " + uri);
            console.log("[+] Backtrace:\n" + Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new()));
            return this.openInputStream(uri);
        };
        console.log("[+] Hooked: ContentResolver.openInputStream()");
    } catch(e) {
        console.log("[-] Error hooking ContentResolver.openInputStream: " + e);
    }

    console.log("[*] === All intent result hooks are set. Waiting for operations... ===");
});
EOF
```

запуск

```q
frida -U -f com.evgeniy.meetway -l /Users/evgeniy/Desktop/hook_intent_results.js
```

тест на приложении [[0_MeetWay]]

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

вижу PID своего приложение

12566   MeetWay
```

----

результаты запуска

```q

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
[KB2003::com.evgeniy.meetway ]-> [*] === Intent Result Hook Script Started ===
[+] Hooked: Activity.startActivityForResult()
[+] Hooked: ActivityResultCallback.onActivityResult()
[+] Hooked: Intent.getData()
[+] Hooked: Intent.getExtras()
[+] Hooked: Intent.getClipData()
[+] Hooked: ContentResolver.query()
[+] Hooked: ContentResolver.openInputStream()
[*] === All intent result hooks are set. Waiting for operations... ===
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getData() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getData() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getClipData() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getData() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getExtras() called -> null
[+] Intent.getData() called -> null
[+] Intent.getClipData() called -> null
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=444]
[+] Intent.getData() called -> yx547697fe6e9d46ea9a7e538922d1425d://auth#access_token=y0__wgBEIbB-MkFGNWYOSD8wJnWF-V774F-fDgMd8y_5PZsoQhxMQDr&token_type=bearer&expires_in=27487803&scope=login%3Aemail%20login%3Ainfo%20login%3Aavatar&cid=j0gcut589qrtqaehn3b27z5970
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=444]
[+] startActivityForResult() called
[+] Request code: -1
[+] Intent: Intent { flg=0x10008000 cmp=com.evgeniy.meetway/.MainActivity (has extras) }
[+] Action: null
[+] Intent.getData() called -> null
[+] Data: null
[+] Backtrace:
java.lang.Exception
	at android.app.Activity.startActivityForResult(Native Method)
	at androidx.activity.ComponentActivity.startActivityForResult(ComponentActivity.kt:683)
	at android.app.Activity.startActivity(Activity.java:6180)
	at android.app.Activity.startActivity(Activity.java:6147)
	at com.evgeniy.meetway.AuthRedirectActivity.navigateToMainActivity(AuthRedirectActivity.kt:127)
	at com.evgeniy.meetway.AuthRedirectActivity.navigateToMainActivity$default(AuthRedirectActivity.kt:108)
	at com.evgeniy.meetway.AuthRedirectActivity$processCallback$1.invokeSuspend(AuthRedirectActivity.kt:90)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:101)
	at android.os.Handler.handleCallback(Handler.java:958)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loopOnce(Looper.java:257)
	at android.os.Looper.loop(Looper.java:368)
	at android.app.ActivityThread.main(ActivityThread.java:8839)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:572)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1049)

[+] Intent.getClipData() called -> null
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getData() called -> null
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getData() called -> null
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
[+] Intent.getExtras() called -> Bundle[mParcelledData.dataSize=44]
```

Самая важная находка-


```
[+] Intent.getData() called -> yx547697fe6e9d46ea9a7e538922d1425d://auth#access_token=y0__wgBEIbB-MkFGNWYOSD8wJnWF-V774F-fDgMd8y_5PZsoQhxMQDr&token_type=bearer&expires_in=27487803&scope=login%3Aemail%20login%3Ainfo%20login%3Aavatar&cid=j0gcut589qrtqaehn3b27z5970
```

Приложение получило `access_token` через неявный интент. Это токен, который используется для аутентификации

---------

#### че мы имеем сейчас

- Приложение получает токен через неявный интент (это нормально для OAuth)
   
- Приложение явно передает токен в `MainActivity` через `Intent` с флагом `cmp=com.evgeniy.meetway/.MainActivity` - это явный интент, что безопасно
--------

------------


##  Результат анализа

### 1. Проверяет ли приложение, что токен действительно от Яндекса?

**Да, частично — есть server-side валидация, но нет local-side:**

| Что проверяет | Статус | Детали |
|---|---|---|
| WebView перехватывает только URL с `oauth.yandex.ru` | ⚠️ Слабо | `contains("oauth.yandex.ru")` — поддельный URL может содержать эту строку |
| `getUserInfoFromYandex(accessToken)` — проверка токена на серверах Яндекса | ✅ Надёжно | GET `https://login.yandex.ru/info` с `Authorization: OAuth <token>` — если токен невалидный, API вернёт ошибку |
| Cloud Function дополнительно валидирует токен | ✅ Надёжно | Серверная проверка перед выдачей JWT |
| `state` параметр (CSRF защита) | ❌ **ОТСУТСТВУЕТ** | В OAuth запросе нет `state` — уязвимость CSRF в OAuth flow |
| `nonce` / challenge | ❌ **ОТСУТСТВУЕТ** | Нет дополнительной защиты |

**Вердикт:** Токен проверяется на сервере Яндекса - подделать его нельзя. Но отсутствие `state` параметра делает OAuth flow уязвимым для CSRF-атак

### 2. Передаётся ли токен через неявные интенты?

**Да, и это критично.**

```
открываем манифест и видим:

<activity

    android:name="com.evgeniy.meetway.AuthRedirectActivity"
    android:exported="true">     ← 👉 ЭКСПОРТИРОВАНА!
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <data android:scheme="yx547697fe6e9d46ea9a7e538922d1425d"
              android:host="auth"/>
              
    </intent-filter>
    
</activity>

```

**Весь OAuth callback приходит через неявный интент**, и его может перехватить любое приложение:

| Угроза                                                                                                                         | Риск                                                                                                     |
| ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| **Перехват callback'а** - любое приложение регистрирует такой же `intent-filter` на схему `yx547697fe6e9d46ea9a7e538922d1425d` | 🔴 **Высокий** - злоумышленник может перехватить `access_token` после авторизации пользователя           |
| **Фальшивый callback** - любое приложение отправляет `yx547697fe6e9d46ea9a7e538922d1425d://auth#access_token=FAKE`             | 🟡 Средний - `getUserInfoFromYandex(FAKE)` вернёт ошибку, но может вызвать DoS или неожиданное поведение |
| **Токен в Intent.getData()** - фрагмент с токеном доступен через `getData()`                                                   | 🟡 Средний - сторонние приложения могут прочитать Intent, если зарегистрируют такой же intent-filter     |

### 3. Куда уходит токен дальше

```
Yandex OAuth → AuthRedirectActivity (implicit intent, exported=true)
  → YandexAuthService.handleCallback()
    → getUserInfoFromYandex(accessToken)  ← проверка на сервере Яндекса
    → Cloud Function (yandexToken)         ← получение JWT
      → SecureStorage.saveJwt(jwt)         ← сохранение JWT локально
      → MainActivity (explicit intent)     ← только auth_success/error, без токена
```

Токен НЕ передаётся через неявные интенты дальше по приложению - все дальнейшие переходы явные

###  Итоговая таблица

| Вопрос                                     | Ответ                                                              |
| ------------------------------------------ | ------------------------------------------------------------------ |
| Проверяет ли приложение источник токена?   | ✅ Да (через API Яндекса + Cloud Function)                          |
| Есть ли `state` параметр в OAuth?          | ❌ **Нет** - CSRF уязвимость                                        |
| `AuthRedirectActivity` exported?           | ❌ **Да** (`exported="true"`) - должен быть `false` с явным вызовом |
| Перехватываем ли callback?                 | ❌ Да - любое приложение может зарегистрировать ту же схему         |
| Токен уходит через неявные интенты дальше? | ✅ **Нет** - дальше только explicit Intents                         |

---

##  вывод: 

**Тест MASTG-TEST-0375 ПРОВАЛЕН.*

Два нарушения:

1. **`AuthRedirectActivity` экспортирована** (`exported="true"`) и принимает неявные интенты - токен может быть перехвачен другим приложением
2. **Отсутствует параметр `state`** в OAuth запросе - CSRF уязвимость

**Рекомендации:**
```kotlin
// 1. Сделать AuthRedirectActivity НЕ экспортированной:
android:exported="false"

// 2. Добавить state параметр в OAuth запрос:
.appendQueryParameter("state", generateCsrfToken())
```

**Фактический риск:** средний - требует установки вредоносного приложения на устройство жертвы

1. **Проверить `AndroidManifest.xml`.** Найди объявление активности, которая обрабатывает OAuth-колбэк. В моем отчете видно, что это `AuthRedirectActivity`. Посмотрим на ее `intent-filter`. Должна быть точно указана схема `yx547697fe6e9d46ea9a7e538922d1425d` и хост `auth` . Это уже хорошо - так приложение не будет обрабатывать другие URI-схемы
   
2. **Проверить нужно -  проверку `redirect_uri`.** В коде OAuth-обработчика должно быть сравнение входящего URI с тем, что был передан в запросе, или с зарегистрированным в настройках приложения. Яндекс рекомендует сверять `redirect_uri` при получении ответа
3. Если эта проверка есть - злоумышленник не сможет подменить адрес доставки
   
4. **Проверить использование явного интента.** Критический момент: после получения колбэка, `AuthRedirectActivity` не должна отправлять токен через `Intent.ACTION_VIEW` или другие неявные интенты. Она должна передать его в `MainActivity` через явный интент. Исследование Facebook SSO подтверждает, что так и делается: активность Яндекса (`FB_client`) возвращает результат в вызывающее приложение через `startActivityForResult` . В твоем случае `AuthRedirectActivity` является _принимающей_ стороной, поэтому проверь, как она передает данные дальше
   
Если схема строго определена, а токен передается через явный интент (как видно в твоих логах), то приложение, скорее всего, корректно идентифицирует источник

---------
###  1: Intent-filter проверен

файл: `AndroidManifest.xml`

```xml
<activity android:name="com.evgeniy.meetway.AuthRedirectActivity"
          android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <data android:scheme="yx547697fe6e9d46ea9a7e538922d1425d"
              android:host="auth"/>
    </intent-filter>
</activity>
```

**Результат:**
- ✅ Схема и хост указаны **точно** - `yx547697fe6e9d46ea9a7e538922d1425d://auth`
- ✅ Другие URI-схемы не пройдут - это хорошо
- ❌ **`exported="true"`** - активность доступна любому приложению на устройстве

---

###  2: Проверка redirect_uri

Код: `YandexAuthService$handleCallback$2.invokeSuspend()`

**Проверка redirect_uri - ОТСУТСТВУЕТ*

Входящий URI никак не сравнивается с `REDIRECT_URI`:

```java
// ❌ НЕТ ПРОВЕРКИ - сразу парсится fragment:
fragment = this.$uri.getFragment();      // берёт fragment из ЛЮБОГО URI
params = fragment.split("&");
accessToken = params.get("access_token");
```

**Результат:**
- ❌ **Сравнения нет** - ни `.equals()`, ни `.startsWith()`, ни `.toString().contains()`
- ❌ Входящий URI не валидируется - `handleCallback()` обработает любой URI, который попадёт в Activity
- ❌ Яндекс рекомендует сверять `redirect_uri`, но в коде этого нет

---

###  3: Явный интент для передачи токена

Код: `AuthRedirectActivity.navigateToMainActivity()`

```java
public final void navigateToMainActivity(Boolean success, String errorMessage) {
    Intent intent = new Intent(this, (Class<?>) MainActivity.class);   // ✅ ЯВНЫЙ
    intent.setFlags(268468224);  // FLAG_ACTIVITY_CLEAR_TOP | SINGLE_TOP

    if (success == true) {
        intent.putExtra("auth_success", true);     // ✅ только boolean
    } else if (success == false) {
        intent.putExtra("auth_error", errorMessage); // ✅ только строка ошибки
    }
    startActivity(intent);  // ✅ не startActivityForResult
    finish();
}
```

**Результат:**
- ✅ **Явный интент** - указан `MainActivity.class`
- ✅ **Токен НЕ передаётся** в Intent - только `auth_success` (boolean) или `auth_error` (String)
- ✅ JWT уже сохранён в `SecureStorage` до вызова `navigateToMainActivity`
- ✅ `ACTION_VIEW` и другие неявные интенты не используются

---

###  сводка по всем пунктам

| Пункт | Результат |
|---|---|
| **1.** Intent-filter с точной схемой | ✅ Схема и хост указаны |
| **1.** `exported="true"` | ❌ Должно быть `false` |
| **2.** Проверка `redirect_uri` в коде | ❌ **Отсутствует** |
| **2.** Параметр `state` в OAuth | ❌ **Отсутствует** |
| **3.** Явный интент в MainActivity | ✅ Explicit, без токена |
| **3.** Токен перехватываем через `getData()` | ⚠️ Да, через фрагмент URI |

---

##  вывод: тест полностью провапленн 

Тест MASTG-TEST-0375 **провален** - два критических нарушения:

1. **`AuthRedirectActivity` экспортирована** - любой может перехватить OAuth callback с `access_token`
2. **Нет проверки `redirect_uri` и `state`** - CSRF уязвимость в OAuth flow


**необходимое исправление:**
```xml
<!-- AndroidManifest.xml -->
<activity android:name="AuthRedirectActivity"
          android:exported="false"/>   <!-- 1 -->

<!-- AppConfig.kt / YandexAuthService.kt -->
.appendQueryParameter("state", secureRandomToken) // 2

// В handleCallback:
if (!incomingUri.toString().startsWith(REDIRECT_URI)) {
    throw SecurityException("Invalid redirect_uri")
}
```


------------





