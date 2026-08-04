# MASTG-TEST-0282: Unsafe Custom Trust Evaluation
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0282/

## что проверяет этот тест

Тест MASTG-TEST-0282 проверяет, не реализован ли в приложении **кастомный `X509TrustManager`** с небезопасной проверкой сертификатов. Если `checkServerTrusted()` переопределён так, что принимает любые сертификаты (пустое тело, всегда return, нет исключений) – MITM-атака становится тривиальной: любой самоподписанный сертификат будет принят.

## какие инструменты использую

беру meetway.apk, через jadx-gui ищу `X509TrustManager` и `checkServerTrusted`:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx (Ctrl+Shift+F) ищу:
1. `X509TrustManager` – реализация кастомного TrustManager
2. `checkServerTrusted` – метод проверки цепочки сертификатов
3. `TrustManagerFactory` – фабрика TrustManager'ов

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Text Search ищу `X509TrustManager`, `checkServerTrusted`, `TrustManagerFactory`**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `X509TrustManager` | ❌ нет |
| `checkServerTrusted` | ❌ нет |
| `TrustManagerFactory` | ❌ нет |
| Кастомный TrustManager | ❌ нет |

Приложение использует стандартный TrustManager, который делегирует проверку сертификатов системе через NSC (см. MASTG-TEST-0286). OkHttp по умолчанию использует системный `TrustManager`.

## вывод

**Тест пройден ✅**

Причина: кастомный TrustManager отсутствует. Валидация сертификатов выполняется через стандартный системный механизм.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
