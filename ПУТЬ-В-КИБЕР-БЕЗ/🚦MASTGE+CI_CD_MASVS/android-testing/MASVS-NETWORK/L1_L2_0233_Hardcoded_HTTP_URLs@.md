# MASTG-TEST-0233: Hardcoded HTTP URLs
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0233/

## что проверяет этот тест

Тест MASTG-TEST-0233 проверяет, есть ли в коде приложения **жёстко зашитые HTTP URL** (не HTTPS). Если приложение использует `http://` для отправки данных – трафик идёт в открытом виде. Любой MITM может перехватить пароли, токены, сообщения без необходимости обходить TLS.

Важно: HTTP URL может быть просто константой (не используется для запросов), или открываться в браузере (там уже не зона ответственности приложения). Тест считается проваленным только если HTTP URL реально используется для сетевого обмена.

## какие инструменты использую

беру meetway.apk, декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx ищу:
1. `http://` во всех декомпилированных классах
2. Конкретно в `com.evgeniy.meetway` – только код приложения
3. Проверяю, используется ли найденный URL для HTTP-запросов или это ссылка в браузер

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – через Text Search (Ctrl+Shift+F) ищу `http://`:**
  - смотрю все вхождения в `sources/com/evgeniy/meetway/` – только код приложения
  - игнорирую `https://` и `schemas.android.com` (XML namespace)

**шаг 3 – анализирую контекст каждого найденного HTTP URL:**
  - это просто строка/константа?
  - используется как URL для OkHttp/HttpURLConnection?
  - открывается в браузере через Intent.ACTION_VIEW?

## что нашёл

**В коде `com.evgeniy.meetway` найден 1 HTTP URL:**

| URL | Где | Тип использования | Статус |
|--|--|--|--|
| `http://empretradingsupport.tilda.ws` | `RegistrationScreenKt.java` строка 184 | Intent.ACTION_VIEW – открывает в браузере | ✅ не опасно |

**Все API-эндпоинты приложения – строго HTTPS:**

| Эндпоинт | URL |
|--|--|
| Cloud Functions | `https://functions.yandexcloud.net/d4efavboiige7c7leqkf/` |
| Yandex Object Storage | `https://storage.yandexcloud.net` |
| Yandex OAuth | `https://oauth.yandex.ru/authorize` |
| Yandex Login Info | `https://login.yandex.ru/info?format=json` |
| Firebase Identity Toolkit | `https://identitytoolkit.googleapis.com` |
| Firebase Firestore | `https://firestore.googleapis.com` |
| YC Notifications | `https://notifications.yandexcloud.net` |

## вывод

**Тест пройден ✅**

Причина: единственный HTTP URL (`http://empretradingsupport.tilda.ws`) открывается в системном браузере через Intent.ACTION_VIEW – это не сетевой запрос из приложения. Все реальные API-эндпоинты используют HTTPS. Cleartext-трафика в коде приложения нет.

Уровень теста: L1, L2.
Профиль: MASVS-NETWORK.

---
