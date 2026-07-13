# MASTG-TEST-0207: Runtime Storage of Unencrypted Data in the App Sandbox
https://mas.owasp.org/MASTG/tests/android/MASVS-STORAGE/MASTG-TEST-0207/

**ГЛАВНЫЙ СМЫСЛ ТЕСТА = не хранит ли приложение чувствительные данные (токены, пароли, личную информацию) в открытом виде или в слабо закодированном виде внутри своей песочницы (`/data/data/...`)**

динамически нужно смотреть - что именно приложение записывает в свое внутреннее хранилище (App Sandbox), и самое главное - **в каком виде**. Даже если данные лежат в защищенной папке приложения, они могут быть там в открытом тексте, что является уязвимостью

---

тест проводится так: 

кидаю шел

```q
adb shell
su

тут же внутри шел копирую

cp -r /data/data/com.evgeniy.meetway /sdcard/meetway_data_before
```

 делаю копию папки приложения с телефона на макбук!
новый терминал
```q
adb pull /sdcard/meetway_data_before ~/Desktop/app_data_before


```




теперь !!!!!! , запускаю приложение и все-все там нажимаю + ввод паролей итп




и снова делаю копию папки

```q
adb shell
su

тут же внутри шел копирую

cp -r /data/data/com.evgeniy.meetway /sdcard/meetway_data_after
```

 делаю копию папки приложения с телефона на макбук!
новый терминал
```q
adb pull /sdcard/meetway_data_after ~/Desktop/app_data_after


```

а теперь нужно глянуть разницу

может чтото новое появилось

```q
diff -r ~/Desktop/app_data_before/ ~/Desktop/app_data_after/
```

------

# ☠️🔴☠️

результаты сравнения: - ПРОСТО БОМБА !!! 

<img src="../../../assets/Снимок20263452070300.18.20.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





### во первых - тут JWT  ☠️


В файле `meetway_prefs.xml` обнаружен JWT-токен

В файле `meetway_secure_prefs.xml` обнаружены ключи и значения, закодированные 


```q
diff -r ~/Desktop/app_data_before/ ~/Desktop/app_data_after/
diff -r /Users/evgeniy/Desktop/app_data_before/shared_prefs/meetway_prefs.xml /Users/evgeniy/Desktop/app_data_after/shared_prefs/meetway_prefs.xml
3c3
<     <string name="internalJWT">eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ5YW5kZXhfaWQiOiIxNDk3MjQzNzgyIiwibG9naW4iOiJzb2FwMjIyMjIyIiwiZW1haWwiOiJzb2FwMjIyMjIyQHlhbmRleC5ydSIsIm5hbWUiOiLQldCy0LPQtdC90LjQuSDQp9C10YDQvdC40LrQvtCyIiwiaWF0IjoxNzgzMDA2NjkwLCJleHAiOjE3ODMwOTMwOTB9.ygBm7zyzG0UpPGdt2loiKiWHDtY7RcIXAOYtyEiiNuM</string>
---
>     <string name="internalJWT">eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ5YW5kZXhfaWQiOiIxNDk3MjQzNzgyIiwibG9naW4iOiJzb2FwMjIyMjIyIiwiZW1haWwiOiJzb2FwMjIyMjIyQHlhbmRleC5ydSIsIm5hbWUiOiLQldCy0LPQtdC90LjQuSDQp9C10YDQvdC40LrQvtCyIiwiaWF0IjoxNzgzMDE5NjM4LCJleHAiOjE3ODMxMDYwMzh9.Dofj1zFHpBBYRyHd0QdBsh7pk4sXk_cAvaz8bob2qlg</string>
4a5
>     <string name="themeColorUser">yellow</string>
diff -r /Users/evgeniy/Desktop/app_data_before/shared_prefs/meetway_secure_prefs.xml /Users/evgeniy/Desktop/app_data_after/shared_prefs/meetway_secure_prefs.xml
2a3
>     <string name="AQy0fklPoz7TYZqh14trzYbxmdlU5z3qtA==">ATiDDlAmu9+TEk2PeVBdiydGHbTzXKdLC1yeyQqkcWj0K0LHEl43mazY7QpayHQo4e8x</string>
4,5d4
<     <string name="AQy0fkndIcXVDWTIurTzCZfdDE8NM59A2v4jxM0=">ATiDDlBSwk8NbmRoAKh3XQ80mMkYVmgNyLDj5X2tNZlghohjV59xfpI2B2CdJ+pKChL1</string>
<     <string name="AQy0fklPoz7TYZqh14trzYbxmdlU5z3qtA==">ATiDDlATjS2SVH/7v2Ev3hAT6uaFey8XjbmYUmWCUi3176tcJ1hnX4zl+mZ7tqYFq0o/</string>
6a6,7
>     <string name="AQy0fkndIcXVDWTIurTzCZfdDE8NM59A2v4jxM0=">ATiDDlCwB0ecP0tuaROBLwh77Eb+ZzvtN1kxu2/F8Zd77LvbHvRtptCtPb6CTswLZV+/</string>
>     <string name="AQy0fkk1GVVPjy3XlFr7vXS07xrSYnB82EEzTda8sq4=">ATiDDlBH0iqwNIeq8f1cf7Xx/ekBVE76vJxEbP/wDMIcAs/vwKjnSAhrrQWMYdnPPEUNkCr7QG6P478Ydg5YEiGqRqo5NwbTtUFYUX3W+p6hwXFzDIA+evSxMenerzfjhegjid5WjBtGiNzuW1XZmXKQyT82KlTYm/Iuw0lfAljuxT024Bwply96inxCd7aNHRz0ooPSzdtvvVtcRxLH/OuvrmJmuIlcarJZDqeC4NU5CMFzhUFEv1U+xYejU76eEw1KDe/6/PChskYzlGKPrDbm4Wf/DWzg1tsQSuibZhIpiJ08pBrGT5/Hjb8adWi0G+E0VerLIU4m6DiH9RAMResiBA+hgjk2+FgMKzTPtb6EIPQnOenoIYVHEQOPDU44p6oG6iMflhkfCutMK4k49D8cmH88+6a4uLiAn2NzHLcveqn+hWc=</string>
```

-------

### вывод

тест полностью провален!!

----

### ПОПРОСИЛ LLM агента open claw разобраться в этом вопросе:





### почему JWT оказался в открытом виде (анализ кода)

нашёл причину — в коде есть **дублирование** сохранения JWT в два разных места

в двух активити JWT сохраняется дважды:
- один раз в `SecureStorage` (зашифрованное хранилище) — это правильно
- второй раз в `meetway_prefs` (обычные SharedPreferences) — через `putString("internalJWT", jwt)`

**YandexAuthWebViewActivity.kt (строки 148-150):**
```kotlin
SecureStorage.saveJwt(jwt)                                // в шифрованное — ок
getSharedPreferences("meetway_prefs", MODE_PRIVATE).edit()
    .putString("internalJWT", jwt).apply()                 // в открытое — УТЕЧКА
```

**AuthRedirectActivity.kt (строки 65-69):**
```kotlin
SecureStorage.saveJwt(jwt)                                // в шифрованное — ок
getSharedPreferences("meetway_prefs", MODE_PRIVATE).edit()
    .putString("internalJWT", jwt).apply()                 // в открытое — УТЕЧКА
```

комментарий в коде всё объясняет:
```kotlin
// Сохраняем JWT в UserDefaults для ViewModel (iOS эквивалент)
```

это баг портирования с iOS. там была связка:
- Keychain — для реального хранения
- UserDefaults — для быстрого доступа из UI (в iOS UserDefaults тоже не шифруется, но там код проверки подписи)

при переносе на Android разработчик:
- сделал `SecureStorage` через EncryptedSharedPreferences (аналог Keychain) — ✅
- но продублировал JWT в обычные SharedPreferences "для ViewModel" — ❌

вторая копия JWT лежит в открытом виде в файле `/data/data/com.evgeniy.meetway/shared_prefs/meetway_prefs.xml`

----

### почему изменились значения в meetway_secure_prefs.xml

это **зашифрованное** хранилище (EncryptedSharedPreferences). изменения там — это норма:
- обновился JWT (новый токен с новым expiration)
- добавилась тема оформления (`themeColorUser = yellow`)
- перезаписались зашифрованные значения

с ними всё в порядке — они зашифрованы AES-GCM, ключ в Android KeyStore

----

### итоговая картина

| файл | шифрование | что внутри | статус |
|---|---|---|---|
| `meetway_prefs.xml` | ❌ нет | JWT в открытом виде, themeColor | 🔴 УТЕЧКА |
| `meetway_secure_prefs.xml` | ✅ AES-GCM | зашифрованные ключи и значения | 🟢 ок |

**корень проблемы:** код сохраняет JWT в два места — SecureStorage (правильно) и обычные SharedPreferences (неправильно). второй save — лишний, от него надо избавиться

------

##### я, как тот разработчик, который портировал код с ios на андроид 😅, подтверждаю выводы LLM агента!