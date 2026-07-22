https://mas.owasp.org/MASTG/tests/android/MASVS-RESILIENCE/MASTG-TEST-0338/
# MASTG-TEST-0338: References to Storage Integrity Check APIs

тест проверка того - **проверяет ли приложение целостность данных, которые оно хранит на устройстве или нет..**

----------


Тест проверяет, использует ли приложение криптографические методы (HMAC, цифровые подписи, хеши) для проверки того, что его локальные данные (файлы, настройки `SharedPreferences`, базы данных) не были изменены извне

**Пример атаки:** Приложение хранит в `SharedPreferences` флаг `is_premium = false`. Злоумышленник на рутованом телефоне изменяет этот файл на `is_premium = true`. Если приложение не проверяет целостность этого файла, оно поверит поддельному флагу и откроет пользователю премиум-доступ. // Тест проверяет, есть ли в коде защита от такого сценария  \\

---------

### как выполняется тест:

открыть апк в JADX-GUI

и через глобальный поиск  (`Ctrl+Shift+F` или `Cmd+Shift+F`) = поиск ключевых слов, указывающих на проверку целостности данных

**Ключевые слова для поиска:**

- `javax.crypto.Mac` – класс для вычисления HMAC.
    
- `java.security.Signature` – класс для цифровых подписей.
    
- `java.security.MessageDigest` – класс для хешей (MD5, SHA).
    
- `doFinal` – метод, который финализирует вычисление хеша или HMAC.
    
- `verify` – метод проверки подписи.
    
- `equals` – часто используется для сравнения вычисленного HMAC с сохраненным.
    
далее анализ найденных мест:

- **Тест пройден:** если найдено в коде, что приложение вычисляет HMAC или подпись для данных, которые сохраняет локально, и проверяет их перед использованием. Это означает, что целостность данных защищена! ✅
    
- **Тест провален:** если нет никаких признаков проверки целостности для локальных данных, или есть использование криптографии, но не для защиты локального хранилища. Это означает, что приложение уязвимо к атакам на локальные данные ❌

-----

приступаю к тесту, открываю apk в jadx

####  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```


и выполняю поиск по ключевымсловам:

```c

- `javax.crypto.Mac` - класс для вычисления HMAC
    - найдено 32 место - где вызывается класс javax.crypto.Mac!
    
    
    
    
- `java.security.Signature` - класс для цифровых подписей
    - найдено 10 мест - где вызывается класс java.security.Signature!
    
    
    
    
    
- `java.security.MessageDigest` - класс для хешей (MD5, SHA)
    - найдено примерно  40 мест - где вызывается класс  java.security.MessageDigest!!!
    
    

    
    
- `doFinal` - метод, который финализирует вычисление хеша или HMAC
    - примерно в 50 местах он вызывается!
    
    
    
    
    
- `verify` - метод проверки подписи
        - примерно в 45 местахон вызывается!
    
    
    
    
    
- `equals` - часто используется для сравнения вычисленного HMAC с сохраненным
     - примерно в 50 местах  вызывается!
```


можно сказать, что все эти методы используются в приложении активно и во многих местах!
теперь нужно проанализировать каждое место и понять, что  и как там работает!

-------

для ускорения процесса , попрошу LLM open claw создать отчет по каждому из найденных мной мест! и далее и сам перепроверю его заключения!

---------

ниже ответ open claw :
(генерация ответа была 3 минуты)

---

---

## Сводка по ключевым словам в коде приложения (com.evgeniy.meetway)

| Ключевое слово                | Всего вхождений (по JADX) | В коде MeetWay | Для integrity хранилища? |
| ----------------------------- | ------------------------- | -------------- | ------------------------ |
| `javax.crypto.Mac`            | ~32                       | 1 файл         | ❌ Нет                    |
| `java.security.Signature`     | ~10                       | 0 файлов       | ❌ Нет                    |
| `java.security.MessageDigest` | ~40                       | 3 файла        | ❌ Нет                    |
| `doFinal`                     | ~50                       | 4 вызова       | ❌ Нет                    |
| `.verify(`                    | ~45                       | 0 файлов       | ❌ Нет                    |
| `.equals(`                    | ~50                       | ~50 (все UI)   | ❌ Нет                    |

---

##  АНАЛИЗ КАЖДОГО НАХОЖДЕНИЯ

---

### 1️⃣ `javax.crypto.Mac` – 32 вхождения всего, 1 в app-коде

#### ✅ Анализируемый файл: `ObjectStorageService.java`
**Путь:** `com.evgeniy.meetway.service.ObjectStorageService.java`
**Строка 30:** `import javax.crypto.Mac;`

**Методы:**

**`hmacSha256Hex(byte[] key, byte[] data)`*** ~строка 1410*
```java
Mac mac = Mac.getInstance("HmacSHA256");
mac.init(new SecretKeySpec(key, "HmacSHA256"));
byte[] bArrDoFinal = mac.doFinal(data);
return bytesToHex(bArrDoFinal);
```
**Назначение:** HMAC-SHA256 для AWS Signature V4 (SigV4).
**Используется для:** Подписи HTTP-запросов к Yandex Object Storage (S3-совместимое хранилище).
**Это integrity локального хранилища?** ❌ **НЕТ.** Это криптографическая подпись для сетевых запросов API (авторизация S3), НЕ для проверки целостности данных на устройстве.

**`hmacSha256(byte[] key, byte[] data)`** *~строка 1422*
```java
Mac mac = Mac.getInstance("HmacSHA256");
mac.init(new SecretKeySpec(key, "HmacSHA256"));
byte[] bArrDoFinal = mac.doFinal(data);
return bArrDoFinal;
```
**Назначение:** То же – HMAC-SHA256 для SigV4, возвращает raw bytes.
**Используется в `getSignatureKey()`** для вычисления цепочки ключей:
`kDate → kRegion → kService → kSigning`
**Это integrity локального хранилища?** ❌ **НЕТ.**

**`getSignatureKey(...)`** *~строка 1435*
**Цепочка:** kDate = HMAC("AWS4"+secret, dateStamp) → kRegion = HMAC(kDate, region) → kService = HMAC(kRegion, service) → signingKey = HMAC(kService, "aws4_request")
**Это integrity локального хранилища?** ❌ **НЕТ.** Стандартный AWS SigV4 key derivation.

**`sha256Hex(byte[] data)`** *~строка 1400*
```java
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] bArrDigest = digest.digest(data);
return bytesToHex(bArrDigest);
```
**Назначение:** SHA-256 хеш для CanonicalRequest (SigV4).
**Это integrity локального хранилища?** ❌ **НЕТ.**

**Вердикт:** ❌ Не относится к проверке целостности локального хранилища.

---

### 2️⃣ `java.security.Signature` – 10 вхождений, **0 в app-коде**

Все 10 вхождений находятся ТОЛЬКО в сторонних библиотеках:
- `com.google.android.gms.internal.p002firebaseauthapi.*` – Google Play Services (Firebase Auth)
- `com.google.crypto.tink.*` – Google Tink криптобиблиотека

**В коде MeetWay – НОЛЬ ИСПОЛЬЗОВАНИЙ.**

**Вердикт:** ❌ Не используется для цифровых подписей вообще. Тест FAIL по этому критерию.

---

### 3️⃣ `java.security.MessageDigest` – ~40 вхождений, 3 в app-коде

#### 3.1 `EncryptionUtil.java`
**Путь:** `com.evgeniy.meetway.util.EncryptionUtil.java`
**Строка 7:** `import java.security.MessageDigest;`

**Метод `generateChatKey(String user1, String user2)`** *~строка 45-75*
```java
String rawKey = AppConfig.ENCRYPTION_SALT + passphrase + "_" + user1 + "_" + user2;
byte[] bytes = rawKey.getBytes(Charsets.UTF_8);
for (int i = 0; i < 100000; i++) {
    byte[] bArrDigest = MessageDigest.getInstance("SHA-256").digest(bytes);
    bytes = bArrDigest;
}
byte[] keyBytes = Arrays.copyOf(bytes, 32);
SecretKeySpec key = new SecretKeySpec(keyBytes, "AES");
```
**Назначение:** 100000 итераций SHA-256 для PBKDF2-like key derivation.
**Используется для:** Генерация AES-256 ключа шифрования чата на основе соли + passphrase + ID участников.
**Это integrity локального хранилища?** ⚠️ **КОСВЕННО.** SHA-256 используется для key derivation, но не для верификации целостности хранимых данных. Ключ применяется для AES-GCM шифрования сообщений чата.
**Важный нюанс:** AES-GCM – authenticated encryption. Если злоумышленник изменит зашифрованные данные, GCM расшифровка выдаст AEADBadTagException. **Это обеспечивает integrity для зашифрованных сообщений.*

❗ **НО** это не HMAC/подпись для хранилища в целом. Защищены только сообщения чата, но НЕ SharedPreferences, НЕ файлы, НЕ базы данных.

#### 3.2 `SslPinningInterceptor.java`
**Путь:** `com.evgeniy.meetway.util.SslPinningInterceptor.java`
**Строка 8:** `import java.security.MessageDigest;`

**Метод `verifyPinningForResponse(Response response)`** *~строка 95-110*
```java
PublicKey publicKey = cert.getPublicKey();
byte[] encoded = publicKey.getEncoded();
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] hashBytes = digest.digest(encoded);
String actualHash = Base64.encodeToString(hashBytes, 2);
```
**Назначение:** SHA-256 хеш публичного ключа сертификата сервера.
**Используется для:** SSL Pinning – проверка, что сертификат сервера соответствует ожидаемому.
**Это integrity локального хранилища?** ❌ **НЕТ.** Это сетевая безопасность (транспортный уровень), не защита данных на устройстве.

#### 3.3 `ObjectStorageService.java` – уже разобрано в п.1
**Метод `sha256Hex()`** – SHA-256 для SigV4 подписи API-запросов.
**Это integrity локального хранилища?** ❌ **НЕТ.**

**Вердикт по MessageDigest:** ❌ Ни одно из 3 app-вхождений не является HMAC/подписью для проверки integrity локального хранилища.

---

### 4️⃣ `doFinal` – ~50 вхождений, 4 в app-коде

#### 4.1 `EncryptionUtil.java:92` – `cipher.doFinal(data)`
```java
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
// ... init in ENCRYPT mode
byte[] encrypted = cipher.doFinal(data);
```
**Назначение:** AES-GCM шифрование сообщения чата.
**Integrity?** ⚠️ AES-GCM включает аутентификацию (AEAD). Если данные изменены – GCM тег не совпадёт.

#### 4.2 `EncryptionUtil.java:126` – `cipher.doFinal(encryptedData)`
```java
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
// ... init in DECRYPT mode
byte[] decrypted = cipher.doFinal(encryptedData);
```
**Назначение:** AES-GCM расшифровка сообщения чата.
**Integrity?** ✅ **ДА, обеспечивает проверку целостности при расшифровке.** Если GCM тест не совпал – вылетит исключение, и decrypt вернёт null.

#### 4.3 & 4.4 `ObjectStorageService.java:1414, 1422` – `mac.doFinal(data)`
**Назначение:** HMAC-SHA256 для подписи S3 запросов (SigV4).
**Integrity?** ❌ НЕ для локального хранилища.

**Вердикт:** ⚠️ `doFinal` в `EncryptionUtil` косвенно защищает целостность сообщений чата через AEAD (GCM). Но это не проверка целостности хранилища как такового.

---

### 5️⃣ `.verify(` – ~45 вхождений, **0 в app-коде**

Все вызовы `.verify()` в библиотеках:
- `com.google.crypto.tink` – внутренние проверки подписей Tink
- `com.google.firebase` – Firebase Auth

**В коде MeetWay – НОЛЬ ВЫЗОВОВ `.verify()`.**

**Вердикт:** ❌ Приложение не использует цифровые подписи для верификации данных.

---

### 6️⃣ `.equals(` – ~50 вхождений, ВСЕ в UI/моделях

Полный список категорий `.equals()` в app-коде:

#### Цвет ячейки (cell color): 10 файлов × 10 цветов
Файлы: `GoalScrinScreenKt.java`, `FirstScrinScreenKt.java`, `GoalScreenUserViewKt.java`, `FirstScreenUserViewKt.java`, `AppColorsKt.java`
```java
if ($this$toCellColor.equals("orange")) { ... }
if ($this$toCellColor.equals("purple")) { ... }
if ($this$toCellColor.equals("yellow")) { ... }
if ($this$toCellColor.equals("red")) { ... }
if ($this$toCellColor.equals("blue")) { ... }
if ($this$toCellColor.equals("cyan")) { ... }
if ($this$toCellColor.equals("gray")) { ... }
if ($this$toCellColor.equals("pink")) { ... }
if ($this$toCellColor.equals("green")) { ... }
if ($this$toCellColor.equals("white")) { ... }
```
**Используется для:** Маппинг названия цвета в Compose Color.
**Это integrity?** ❌ НЕТ.

#### Тип поста: `FeedViewModel.java`
```java
if (!typePost.equals("action")) { ... }
if (typePost.equals("goal")) { ... }
if (typePost.equals("achievement")) { ... }
```
**Используется для:** Фильтрация постов в ленте.
**Это integrity?** ❌ НЕТ.

#### Тип сообщения: `ChatViewModel.java`
```java
if (str.equals("text")) { ... }
if (str.equals(MimeTypes.BASE_TYPE_AUDIO)) { ... }
if (str.equals("image")) { ... }
```
**Используется для:** Определение типа контента в чате.
**Это integrity?** ❌ НЕТ.

**Вердикт:** ❌ Все `.equals()` – UI/бизнес-логика, НИ ОДИН не для сравнения HMAC/хешей.

---

##  ИТОГО

### Тест MASTG-TEST-0338: ❌ **ПРОВАЛЕН**

1. **Нет HMAC для локальных данных** – `Mac` используется только для AWS SigV4 (сетевые запросы), НЕ для проверки целостности SharedPreferences/файлов/БД на устройстве.

2. **Нет цифровых подписей** – `Signature` не используется вообще в коде приложения.

3. **Нет `.verify()`** для проверки подписей или HMAC на локальных данных.

4. **`.equals()` используется только для UI** – цвета ячеек, типы постов, типы сообщений.

### Что ЕСТЬ - защита сообщений и все, но не хранилища

| Компонент | Защита | Примечание |
|---|---|---|
| `SecureStorage` (SharedPreferences) | ✅ **EncryptedSharedPreferences** (AES256-GCM + AES256-SIV) | Библиотека AndroidX – integrity на уровне фреймворка |
| Сообщения чата | ✅ **AES-256/GCM** – authenticated encryption | AEAD обеспечивает integrity при расшифровке |
| JWT токен | ✅ Хранится в EncryptedSharedPreferences | Защищён самой библиотекой |
###  Чего НЕТ (уязвимости):

| Данные                                        | Угроза                                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Файлы кэша (изображения, видео, аудио)        | ❌ Нет HMAC/подписи = можно подменить                                                                            |
| Локальная БД (если есть Room/SQLite)          | ❌ Не обнаружено проверок целостности                                                                            |
| Флаги в SharedPreferences (is_premium и т.п.) | ❌ Нет кастомной HMAC-верификации (но EncryptedSharedPreferences спасает, т.к. используется для всего хранилища) |

### как исправить косяки:

Для полноценного прохождения теста MASTG-TEST-0338 рекомендуется:

1. **HMAC для файлов кэша** - вычислять HMAC-SHA256 для файлов, загруженных из S3, и проверять перед использованием
2. **Подпись shared preferences** - если появятся флаги, управляющие доступом (isPremium, isAdmin), подписывать их HMAC с ключом из SecureStorage
3. **Цифровые подписи** --- добавить `Signature` для верификации критичных конфигурационных данных, получаемых с сервера

---

##  Приложение: Полный список файлов app-кода с крипто-операциями

| Файл | Крипто-операция | Назначение | Integrity хранилища? |
|---|---|---|---|
| `EncryptionUtil.java` | SHA-256 (x100k), AES-256/GCM doFinal | Шифрование чат-сообщений | ❌ (AEAD только для сообщений) |
| `ObjectStorageService.java` | SHA-256, HMAC-SHA256 (Mac), doFinal | AWS SigV4 подпись S3 запросов | ❌ |
| `SslPinningInterceptor.java` | SHA-256 (MessageDigest) | Certificate pinning | ❌ |
| `SecureStorage.java` | EncryptedSharedPreferences (AndroidX) | Шифрование JWT/паролей/ключей | ✅ (на уровне библиотеки) |
| `PinSessionManager.java` | SecureRandom | Генерация PIN-кода | ❌ |

---

----------

перепроверил все что написал мне LLM агент, все реально так ! тем более, я как создатель данного приложения - знаю, что защиту хранилища не внедрял!

------------

## вывод - тест провален!  нет проверки целостности данных при работе приложения!
