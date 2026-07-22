# MASTG-TEST-0217: Insecure TLS Protocols Explicitly Allowed in Code
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0217/

## что проверяет этот тест

Тест MASTG-TEST-0217 проверяет, не разрешает ли код приложения **небезопасные версии TLS** (1.0, 1.1) через:

1. **Java SSLContext** – вызов `SSLContext.getInstance("TLSv1.1")` или `SSLSocket.setEnabledProtocols()`
2. **OkHttp ConnectionSpec** – использование `ConnectionSpec.COMPATIBLE_TLS` или `ConnectionSpec.Builder.tlsVersions()` с TLS 1.0/1.1

Если приложение явно разрешает старые TLS --  MITM может понизить версию протокола и взломать шифрование.

## какие инструменты использую

беру meetway.apk, декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx ищу:
1. `SSLContext.getInstance` – кастомный контекст с указанием протокола
2. `setEnabledProtocols` – выбор протоколов для SSLSocket
3. `ConnectionSpec` – настройки OkHttp
4. `COMPATIBLE_TLS` – разрешает TLS 1.0/1.1 в OkHttp

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – через Text Search (Ctrl+Shift+F) ищу в `com.evgeniy.meetway`:**
  - `SSLContext.getInstance` – есть ли кастомный выбор протокола
  - `ConnectionSpec` – настройки TLS для OkHttp
  - `COMPATIBLE_TLS` – разрешение старых протоколов

**шаг 3     смотрю конфигурацию OkHttpClient в NetworkModule, CloudFunctionService, ObjectStorageService**

## чегоооо нашёл

| Что искал | Нашёл? |
|--|--|
| `SSLContext.getInstance("TLSv...")` | ❌ нет |
| `SSLSocket.setEnabledProtocols` | ❌ нет |
| `ConnectionSpec.COMPATIBLE_TLS` | ❌ нет |
| `ConnectionSpec.Builder.tlsVersions` | ❌ нет |
| Кастомный ConnectionSpec | ❌ нет |

**OkHttpClient в приложении создаётся с настройками по умолчанию:**
- `NetworkModule.createOkHttpClient()` – таймауты + interceptor, без кастомного ConnectionSpec
- `CloudFunctionService` – таймауты + certificatePinner, без кастомного ConnectionSpec
- `ObjectStorageService` – таймауты, без кастомного ConnectionSpec
- `YandexAuthService` – `new OkHttpClient()` – вообще без настроек (дефолт)

**Эндпоинты MeetWay (все – HTTPS, OkHttp по умолчанию TLS 1.2+):**

| Эндпоинт | URL | TLS (OkHttp default) |
|--|--|--|
| Cloud Functions | `https://functions.yandexcloud.net/d4efavboiige7c7leqkf/` | MODERN_TLS (1.2+) ✅ |
| Object Storage | `https://baket-ivaaro.storage.yandexcloud.net` | MODERN_TLS (1.2+) ✅ |
| Yandex OAuth | `https://oauth.yandex.ru/authorize` | MODERN_TLS (1.2+) ✅ |
| Yandex Login Info | `https://login.yandex.ru/info?format=json` | MODERN_TLS (1.2+) ✅ |
| Firebase Firestore | `https://firestore.googleapis.com` | MODERN_TLS (1.2+) ✅ |
| Firebase Identity | `https://identitytoolkit.googleapis.com` | MODERN_TLS (1.2+) ✅ |

OkHttp по умолчанию использует `ConnectionSpec.MODERN_TLS`, который поддерживает TLS 1.2+ и не включает TLS 1.0/1.1.

## вывод

**Тест пройден ✅**

Причина: приложение не использует небезопасные версии TLS ни через SSLContext, ни через кастомные ConnectionSpec OkHttp. Все OkHttpClient создаются с настройками по умолчанию (MODERN_TLS – TLS 1.2+). TLS 1.0/1.1 не разрешены.



---
