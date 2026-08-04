# MASTG-TEST-0234: Missing Implementation of Server Hostname Verification with SSLSockets
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0234/

## что проверяет этот тест

Тест MASTG-TEST-0234 проверяет, используется ли в приложении `SSLSocket` **без `HostnameVerifier`**. В отличие от `HttpsURLConnection` и OkHttp, `SSLSocket` не проверяет hostname автоматически – разработчик должен сам вызвать `HostnameVerifier.verify()`. Если этого не сделать, MITM-атака возможна даже с валидным сертификатом (сертификат подходит к домену злоумышленника, но не к твоему серверу).

Также важно: Network Security Configuration (NSC) **не влияет** на `SSLSocket` – он его попросту не перехватывает.

## какие инструменты использую

беру meetway.apk и декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx ищу:
1. `SSLSocket` – использование низкоуровневых TLS-сокетов
2. `HostnameVerifier` – проверка hostname в коде
3. `SSLContext` – кастомные SSL-контексты

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – через Text Search (Ctrl+Shift+F) ищу в `com.evgeniy.meetway`:**
  - `SSLSocket` – есть ли прямой вызов?
  - `HostnameVerifier` – есть ли реализация?
  - `SSLContext` – есть ли кастомный контекст?

**шаг 3 – если нахожу – проверяю контекст вызова**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `SSLSocket` | ❌ нет |
| `HostnameVerifier` | ❌ нет |
| `SSLContext` | ❌ нет |
| `X509TrustManager` | ❌ нет |

Приложение **не использует** `SSLSocket` вообще. Весь сетевой обмен идёт через **OkHttp** (`CloudFunctionService`, `ObjectStorageService`), который автоматически проверяет hostname при TLS-рукопожатии. Окружение OkHttp контролируется NSC (см. MASTG-TEST-0235).

## вывод

**Тест пройден ✅**

Причина: SSLSocket не используется – hostname verification OkHttp обрабатывает автоматически. Кастомных SSL-контекстов или TrustManager'ов нет.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
