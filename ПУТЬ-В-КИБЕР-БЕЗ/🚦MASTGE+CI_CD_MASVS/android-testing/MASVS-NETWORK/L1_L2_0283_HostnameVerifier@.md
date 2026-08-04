# MASTG-TEST-0283: Incorrect Implementation of Server Hostname Verification
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0283/

## что проверяет этот тест

Тест MASTG-TEST-0283 проверяет, не реализован ли **кастомный `HostnameVerifier`** с небезопасной логикой. Если `verify()` всегда возвращает `true` – приложение принимает сертификаты от любого домена. MITM с сертификатом на другой домен пройдёт.

## какие инструменты использую

беру meetway.apk, через jadx-gui ищу `HostnameVerifier`:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx (Ctrl+Shift+F) ищу:
1. `HostnameVerifier` – реализация кастомного верификатора
2. `verify(...)` – метод проверки hostname
3. `setHostnameVerifier` – установка кастомного верификатора на HttpsURLConnection/OkHttp

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Text Search ищу `HostnameVerifier`, `verify(`, `setHostnameVerifier`**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `HostnameVerifier` (кастомный) | ❌ нет |
| `verify(String, SSLSession)` | ❌ нет |
| `setHostnameVerifier` | ❌ нет |

Приложение использует OkHttp, который имеет встроенный `HostnameVerifier`. Кастомная реализация отсутствует.

## вывод

**Тест пройден ✅**

Причина: кастомный `HostnameVerifier` не реализован. OkHttp обрабатывает проверку hostname автоматически и безопасно.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
