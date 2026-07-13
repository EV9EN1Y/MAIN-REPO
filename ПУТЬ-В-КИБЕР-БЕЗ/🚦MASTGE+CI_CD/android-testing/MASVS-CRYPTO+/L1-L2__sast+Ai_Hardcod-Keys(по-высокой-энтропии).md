# MASTG-TEST-0212: Use of Hardcoded Cryptographic Keys in Code
https://mas.owasp.org/MASTG/tests/android/MASVS-CRYPTO/MASTG-TEST-0212/

-----
Классика!

Тест ищет в коде **жестко закодированные (хардкод)** ключи и пароли, которые используются для шифрования, подписи или аутентификации. Если такие ключи найдены, тест провален, так как злоумышленник сможет их легко извлечь

----

можно искать конечно, по ключевым словам, но это слабо

поэтому, я буду использовать способы поиска строк с высокой энтропией! и уже там проверять все находки!

------

###  KeyHunter 

## умеет работать с APK-файлами напрямую

KeyHunter распакует APK, просканирует все файлы (включая Smali-код и нативные библиотеки) по своей базе правил и найдет все потенциальные ключи, токены, пароли и URL-адреса[](https://github.com/bigzooooz/KeyHunter). Результат будет в папке с отчётом, где ты сможешь спокойно изучить каждую находку


```q
cd ~
git clone https://github.com/0xj3st3r/key-hunter.git
cd key-hunter

зависимости

pip install -r requirements.txt

запуск

python key-hunter.py --apk "/Users/evgeniy/Desktop/apk-meetway-с=защитой-от=отладки=и=взлома/meetway_with_protection.apk" --include-native --format html
```

------

вот отчет

```q
=== Scan complete: meetway_with_protection ===
Unique findings : 18
Total locations : 26
  HIGH      1
  MEDIUM    1
  LOW       6
  INFO      10

Report: /Users/evgeniy/key-hunter/reports/meetway_with_protection_report.html
```

вот так все красиво открывается HTML

<img src="../../../assets/Снимок2026-07-0315.32.45.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





![[meetway_with_protection_report.html]]

```q
# key-hunter report

Target: **meetway_with_protection** · Generated 2026-07-03 15:31:05

Secrets 8

URLs / Endpoints 10

UNIQUE

8

LOCATIONS

9

CRITICAL

0

HIGH

1

MEDIUM

1

LOW

6

AllCriticalHighMediumLow

HIGHGoogle API Keygoogle-api-key2 locations

Google API Key (AIza...)

Value

AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEM

Entropy

5.03 bits/char

Tags

google, cloud

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/resources/res/values/strings.xml:143:35

<string name="google_api_key">AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEM</string>

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/resources/res/values/strings.xml:145:51

<string name="google_crash_reporting_api_key">AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEM</string>

MEDIUMGoogle OAuth Client IDgoogle-oauth-client-id1 location

Google OAuth 2.0 client ID

Value

155577473113-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com

Entropy

3.31 bits/char

Tags

google, oauth

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/resources/res/values/strings.xml:69:42

<string name="default_web_client_id">155577473113-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com</string>

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

([a-zA-Z0-9-!#$%&

Entropy

3.62 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/okhttp3/MediaType.java:23:42

private static final String TOKEN = "([a-zA-Z0-9-!#$%&'*+.^_`{|}~]+)";

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

).append(this.keys[i]).append(

Entropy

3.86 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/io/grpc/PersistentHashArrayMappedTrie.java:143:40

valuesSb.append("(key=").append(this.keys[i]).append(" value=").append(this.values[i]).append(") ");

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

547697fe6e9d46ea9a7e538922d1425d

Entropy

3.55 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/com/evgeniy/meetway/YandexAuthWebViewActivity.java:46:46

private static final String CLIENT_ID = "547697fe6e9d46ea9a7e538922d1425d";

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

722de50c58c346178ebd3fbf6a31132c

Entropy

3.80 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/com/evgeniy/meetway/util/AppConfig.java:21:52

public static final String YANDEX_CLIENT_ID = "722de50c58c346178ebd3fbf6a31132c";

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

YCAJESl8Vc6eod-TSiavKAGVh

Entropy

4.40 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/com/evgeniy/meetway/util/AppConfig.java:16:49

public static final String S3_ACCESS_KEY = "YCAJESl8Vc6eod-TSiavKAGVh";

LOWGeneric Key/Secret Assignmentgeneric-key-value-secret1 location

Variable name suggests a secret and value looks credential-like

Value

mobile-subtype

Entropy

3.52 bits/char

Tags

generic

/Users/evgeniy/key-hunter/meetway_with_protection_decompiled/sources/com/google/android/datatransport/cct/CctTransportBackend.java:74:47

static final String KEY_MOBILE_SUBTYPE = "mobile-subtype";
```


------

## LLM open claw - попрошу его, чтобы он просто просканировал код, попробовал обнаружить в нем хардкод ключи!




----

### вывод

Найденные множества ключей, некоторые из них не критические, но некоторые критически позволяют получить доступ при некоторых обстоятельствах к backend ресурсам, являются полностью категорически критическими уязвимостями!! тест выполнен, тест провален!

-----
ниже - отчет пишет LLM (разрешил ему дописать тут файл)

## Сканирование Скайнет (OpenClaw) – 04.07.2026

Просканированы декомпилированные исходники APK (`meetway_with_protection`) вручную. Найдены следующие хардкод ключи и секреты:

### 1. S3 Object Storage – CRITICAL
Файл: `AppConfig.java`
- **S3_ACCESS_KEY**: `YCCEC3JESCE85R6dod-TSiavKAGVh`  — доступ к Object Storage
- **S3_SECRET_KEY**: `YeWvrgs2XIevrenfJerqeCdfxjrfKtG$FD$7ep2`  — полный доступ к Object Storage
- **S3_ENDPOINT**: `https://storage.cloud.net`
- **S3_BUCKET**: `baket-ivaro`
- **S3_REGION**: `eu-west-1`

Комментарий: связка Access + Secret Key даёт полный доступ к S3-совместимому хранилищу Yandex Cloud. Злоумышленник может читать, загружать и удалять любые файлы.

### 2. Yandex Cloud Functions Endpoint – CRITICAL
Файл: `AppConfig.java` / `CloudFunctionService.java`
- **BASE_URL**: `https://functions.yandexcloud.net/dae8fDcvboipede7c7leqkt/`
- Публичный URL Yandex Cloud Function, все API-вызовы приложения идут через неё. Сам по себе не секрет (публичный эндпоинт), но указание в хардкоде не критично.

### 3. Yandex OAuth Client ID – MEDIUM
Файл: `AppConfig.java`
- **YANDEX_CLIENT_ID**: `722de50c58c346178ebd3fbf6a31132c`
- Дублируется в `YandexAuthWebViewActivity.java` как `CLIENT_ID`: `547697fe6e9d46ea9a7e538922d1425d`
- Разные Client ID? В YandexAuthWebViewActivity используется custom scheme `yx547697fe6e9d46ea9a7e538922d1425d://auth` – это **другой** Client ID (вероятно, для WebView-авторизации)

### 4. Encryption Parameters – HIGH
Файл: `AppConfig.java` / `EncryptionUtil.java`
- **ENCRYPTION_SALT**: длинная строка (107 символов) – соль для PBKDF2-подобной генерации ключей чатов
- **ENCRYPTION_PASSPHRASE_KEY**: `kkr15` – ключ для хранения passphrase в SharedPreferences
- **KEYCHAIN_JWT_KEY**: `internalJWT` – ключ для JWT в хранилище
- **KEYCHAIN_ENC_KEY**: `messageEncryptionKey` – ключ для encryption key в хранилище
- Алгоритм: AES/GCM/NoPadding, IV = 12 байт, tag = 128 бит, iterations = 100000

Комментарий: ENCRYPTION_SALT хардкожена, что ослабляет защиту ключей шифрования сообщений. Если passphrase (kkr15) тоже скомпрометирован – ключи чатов восстанавливаются.

### 5. Google API Keys – MEDIUM
Файл: `strings.xml`
- **google_api_key**: `AIzaSyBYF96ObWXyrvpYIlliH4J07hgcNxKzjEM`
- **google_crash_reporting_api_key**: (тот же ключ)
- **default_web_client_id**: Google OAuth Client ID

### 6. SSL Certificate Pinning – INFO (но хардкод)
Файл: `AppConfig.java` (SslPins) / `NetworkModule.java`
7 хардкоженных SHA256-пинов:
- `api.yandex.ru` × 2 пина
- `storage.yandexcloud.net` × 2 пина
- `functions.yandexcloud.net` × 2 пина (+ бэкап-пин)

### 7. Yandex Notifications Endpoint – INFO
Файл: `AppConfig.java`
- **YC_NOTIFICATIONS_ENDPOINT**: `https://notifications.yandexcloud.net`
- **YC_REGION**: `ru-central1`

### 8. Дополнительные константы
Файл: `EncryptionUtil.java`
- `AES_MODE = "AES/GCM/NoPadding"` – жёстко зашит
- `GCM_IV_LENGTH = 12`, `GCM_TAG_LENGTH = 128`, `ITERATIONS = 100000`

Файл: `YandexAuthWebViewActivity.java`
- `SCOPE = "login:email login:info login:avatar"`
- `REDIRECT_URI = "yx547697fe6e9d46ea9a7e538922d1425d://auth"`

### Резюме от Скайнета

| # | Находка | Риск | Тип |
|:-|:-|:-|:-|
| 1 | S3_ACCESS_KEY + S3_SECRET_KEY | 🔴 CRITICAL | Cloud storage compromise |
| 2 | Yandex Cloud Functions URL | 🟡 INFO | Публичный эндпоинт |
| 3 | Yandex OAuth Client ID (×2) | 🟠 MEDIUM | OAuth client compromise |
| 4 | ENCRYPTION_SALT + params | 🔴 HIGH | Ослабление шифрования чатов |
| 5 | Google API Key | 🟠 MEDIUM | Google services access |
| 6 | SSL Pins (7 шт) | 🟢 INFO | Хардкод, но стандартная практика |
| 7 | YC Notifications endpoint | 🟢 INFO | Публичный эндпоинт |

-----------



**Вердикт:** тест **провален**. Найдено 2+ хардкод секрета, дающих доступ к backend-инфраструктуре (S3-ключи, соль шифрования). Критично: S3_ACCESS_KEY / S3_SECRET_KEY - требуют немедленной ротации и выноса в защищённое хранилище (secrets manager / env vars).

---


### итого - мой вывод - уже LLM конкретно нашел реально все, что есть в проекте!
польза реальная есть!
но использовать нужно с умом, и как дополнение к основным методам!
и конечно, перепроверять все там!


llm работал своими мозгами, то есть сам анализировал код , + сканировал классически через grep

