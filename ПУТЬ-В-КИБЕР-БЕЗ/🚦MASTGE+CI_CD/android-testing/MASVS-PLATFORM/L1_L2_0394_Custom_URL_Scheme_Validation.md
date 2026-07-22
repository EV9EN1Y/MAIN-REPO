# MASTG-TEST-0394: Missing Input Validation in Custom URL Scheme Handlers
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0394/

## что проверяет этот тест

Тест MASTG-TEST-0394 проверяет, валидирует ли приложение **параметры из custom URL scheme** (deep link). Приложение регистрирует свою схему (например, `myapp://`), и когда приходит интент с такой схемой – оно достаёт данные из URI. Если эти данные используются без проверки, злоумышленник может передать любые параметры

```
myapp://transfer?amount=-1            → обход бизнес-логики
myapp://open?path=../../secret.db     → path traversal
myapp://search?q=<script>...</script> → XSS/инъекция
```

В отличие от iOS, Android не показывает, какое приложение отправило Intent – источник неизвестен

## какие инструменты использую

беру meetway.apk, смотрю манифест и код:

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу:
1. `<intent-filter>` с кастомной `android:scheme`
2. Код, который читает `intent.data` – `getData()`, `getQueryParameter()`, `getPathSegment()`
3. Как используются эти данные – есть ли валидация

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui и ищу кастомные схемы:**
  - Вкладка AndroidManifest.xml → ищу `<intent-filter>` с `<data android:scheme>`

**шаг 2 – через встроенный поиск jadx нахожу обработчик deep link:**
  - Text Search (Ctrl+Shift+F) → `handleDeepLink`
  - Text Search (Ctrl+Shift+F) → `getData()`
  - Text Search (Ctrl+Shift+F) → `intent.data`

**шаг 3 – открываю найденный класс в jadx и оцениваю валидацию**

## что нашёл

**Custom URL scheme:** `yx547697fe6e9d46ea9a7e538922d1425d://auth`

**Обработчик – `MainActivity.handleDeepLink()`:**
```kotlin
private fun handleDeepLink(intent: Intent?) {
    val data = intent?.data ?: return
    if (data.scheme == "yx547697fe6e9d46ea9a7e538922d1425d"
        && data.host == "auth") {
        // → передаёт URI дальше в AuthRedirectActivity
    }
}
```

| Проверка в коде | Статус |
|--|--|
| Проверка scheme | ✅ есть |
| Проверка host | ✅ есть |
| Проверка access_token (длина, формат, символы) | ❌ нет |
| Валидация на сервере | ✅ Cloud Function проверяет токен через Яндекс API |

**Главный защитный фактор:** токен отправляется на сервер, и сервер сам проверяет его валидность через Яндекс. Если токен фейковый – JWT не будет сгенерирован, аутентификация не пройдёт



## вывод

**Тест пройден 

Причина: клиент не валидирует access_token на своей стороне, но сервер делает полную проверку через Яндекс. Фейковый токен не пройдёт – JWT не сохранится
Риск минимальный
нормус

