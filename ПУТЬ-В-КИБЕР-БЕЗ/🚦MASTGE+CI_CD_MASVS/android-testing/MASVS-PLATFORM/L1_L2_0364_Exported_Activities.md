# MASTG-TEST-0364: Exported And Unprotected Activities That Expose Sensitive Functionality
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0364/

## что проверяет этот тест

Тест MASTG-TEST-0364 проверяет, не экспортирует ли приложение **Activity** без защиты. Activity с `android:exported="true"` может быть запущена из любого другого приложения на устройстве. Если эта Activity показывает чувствительные данные, меняет настройки, подтверждает действия – злоумышленник может вызвать её напрямую, минуя нормальный флоу приложения (например, авторизацию)

## какие инструменты использую

беру meetway.apk и смотрю AndroidManifest.xml:

```bash
jadx-gui ~/Desktop/meetway.apk
```

в манифесте ищу:
1. Все `<activity>` элементы
2. У каждого проверяю `android:exported`
3. Смотрю `android:permission`
4. Анализирую intent-filter и код activity

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui и просматриваю все `<activity>`:**
  - Вкладка AndroidManifest.xml → ищу элементы `<activity>`
  - У каждого проверяю `android:exported`, `android:permission`, `intent-filter`

**шаг 2 – для каждой exported activity открываю её декомпилированный код в jadx:**
  - Навигация по классу → просмотр onCreate(), intent-handling

## что нашёл

**Все Activity в манифесте:**

| Activity | exported | Что делает | Оценка |
|--|--|--|--|
| `MainActivity` | ✅ true (LAUNCHER) | Главная точка входа, запускает UI | ✅ Норма |
| `AuthRedirectActivity` | ✅ true | Принимает OAuth callback через deep link, отправляет токен на сервер | ⚠️ Есть риск |
| `YandexAuthWebViewActivity` | ❌ false | WebView для входа через Яндекс | ✅ Не экспортирована |

**Анализ `AuthRedirectActivity` (единственная не-LAUNCHER exported):**

Она принимает кастомную схему `yx547697fe6e9d46ea9a7e538922d1425d://auth`. Любое приложение может отправить такой интент с access_token. Но:
1. Токен передаётся в `YandexAuthService.handleCallback()` – Cloud Function на сервере
2. Cloud Function проверяет access_token через API Яндекса
3. Если токен невалидный – сервер возвращает ошибку, JWT не сохраняется

То есть клиент доверяет серверу – это правильная архитектура.

## вывод

**Тест пройден ⚠️**

Причина: `AuthRedirectActivity` принимает токен от любого приложения, но он валидируется на сервере. Подменить токен нельзя – фейковый токен не пройдёт проверку Яндексом. `MainActivity` – стандартная LAUNCHER, норм.

Уровень теста: L1, L2.
Профиль: MASVS-PLATFORM.

---
