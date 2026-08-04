# MASTG-TEST-0218: Insecure TLS Protocols in Network Traffic
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0218/

## что проверяет этот тест

Тест MASTG-TEST-0218 (динамический) проверяет, какая **версия TLS реально используется** при соединениях приложения. Статический анализ (MASTG-TEST-0217) может показать, какие протоколы разрешены в коде, но только перехват трафика покажет, что по факту договорились клиент и сервер.

Сервер может быть старым и форсировать TLS 1.1 даже если клиент поддерживает 1.3 – тест это выявит.

## какие инструменты использую

- **Burp Suite** на Маке – `192.168.0.11:8080`
- **OnePlus 8T** (Android 14, root, Magisk)
- **APEX injection** – `proxy-on.sh` (nsenter bind-mount Conscrypt)
- **Frida** – `ssl-unpin-meetway.js`

## подготовка

APEX injection для обхода Conscrypt APEX (Android 14):

```bash
adb exec-out su -c sh /data/local/tmp/proxy-on.sh
```

Детали: см. `P_dast-burp_данные-не-указанные-в-политике.md`

## как провожу тест

**шаг 1 – запускаю Burp Suite**

**шаг 2 – отключаю прокси, логинюсь в приложении (чтобы Яндекс OAuth прошёл)**

**шаг 3 – включаю прокси + APEX injection**

**шаг 4 – запускаю MeetWay через Frida:**
```bash
frida -U -f com.evgeniy.meetway -l /tmp/ssl-unpin-meetway.js
```

**шаг 5 – в Burp смотрю TLS-версии запросов (Alerts → TLS Info / TLS Handshake)**

## что нашёл

Все перехваченные запросы:

| Эндпоинт | TLS версия (Burp) |
|--|--|
| `functions.yandexcloud.net` | **TLS 1.3** |
| `storage.yandexcloud.net` | **TLS 1.3** |
| `login.yandex.ru` | **TLS 1.3** |

Ни одного TLS 1.0 или 1.1. Все соединения – TLS 1.3 (максимальная защита).

Подтверждает статический анализ: OkHttp по умолчанию использует MODERN_TLS (TLS 1.2+), серверы Yandex Cloud форсят TLS 1.3.

## вывод

**Тест пройден ✅**

Причина: все перехваченные соединения используют TLS 1.3. Insecure TLS версии (1.0, 1.1) не обнаружены ни на клиенте, ни на сервере.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
