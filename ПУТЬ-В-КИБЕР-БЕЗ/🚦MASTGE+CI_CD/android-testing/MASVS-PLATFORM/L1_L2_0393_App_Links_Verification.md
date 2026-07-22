# MASTG-TEST-0393: Use of Unverified App Links
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0393/

## что проверяет этот тест

Тест MASTG-TEST-0393 проверяет, использует ли приложение **Android App Links** с `android:autoVerify="true"`. App Links – это механизм, который позволяет приложению открывать http/https ссылки без запроса у пользователя (без диалога "открыть в браузере или приложении"). Если `autoVerify` не включён, любое другое приложение может зарегистрировать такой же intent-filter и перехватывать ссылки вместо вашего – это фишинг

**Важно:** Атака работает только для http/https схем. Для кастомных схем (myapp://) autoVerify не применяется – они всегда могут быть перехвачены




=-------

## какие инструменты использую

беру meetway.apk, смотрю манифест:

```bash
jadx-gui ~/Desktop/meetway.apk

```

ищу:
1. `<intent-filter>` с `android:scheme="http"` или `https`
2. Атрибут `android:autoVerify="true"`
3. `<data android:scheme>` – какие схемы используются

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui и просматриваю все `<intent-filter>`:**
  - Вкладка AndroidManifest.xml → ищу `<intent-filter>` с `<data android:scheme>`
  - Для http/https проверяю наличие `android:autoVerify="true"`

## что нашёл

**intent-filter в приложении:**

| Activity / Component | scheme | http/https? | autoVerify? |
|--|--|--|--|
| `AuthRedirectActivity` | `yx547697fe6e9d46ea9a7e538922d1425d` | ❌ кастомная | – |
| `MainActivity` | (LAUNCHER) | ❌ нет data scheme | – |
| `YandexAuthWebViewActivity` | – (not exported) | ❌ | – |
| `MeetWayMessagingService` | – (not exported) | ❌ | – |

**http/https deep link-и: НЕ ИСПОЛЬЗУЮТСЯ.** Все схемы кастомные.

## вывод

**Тест пройден ✅**

Причина: http/https deep link-и не используются, autoVerify не требуется. Все deep link-и через кастомную схему Yandex OAuth.








э
---
