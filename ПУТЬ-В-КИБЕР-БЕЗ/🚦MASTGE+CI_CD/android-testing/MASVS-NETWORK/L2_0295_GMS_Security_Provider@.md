# MASTG-TEST-0295: GMS Security Provider Not Updated
https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/MASTG-TEST-0295/

## что проверяет этот тест

Тест MASTG-TEST-0295 проверяет, обновляет ли приложение **Security Provider** через Google Play Services. Security Provider содержит реализации криптографических алгоритмов (TLS, шифрование). Если он не обновлён – на устройстве могут остаться уязвимые версии (например, в старых версиях Provider были баги в TLS-реализации).

Обновление делается через `ProviderInstaller.installIfNeeded()` или `installIfNeededAsync()` – желательно при первом запуске, до любых сетевых соединений.

## какие инструменты использую

беру meetway.apk, через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx (Ctrl+Shift+F) ищу:
1. `ProviderInstaller` – класс для обновления Security Provider
2. `installIfNeeded` – синхронное обновление
3. `installIfNeededAsync` – асинхронное обновление
4. `GoogleApiAvailability` – альтернативные проверки

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – Text Search ищу `ProviderInstaller`, `installIfNeeded`, `SecurityProvider`**

## что нашёл

| Что искал | Нашёл? |
|--|--|
| `ProviderInstaller.installIfNeeded()` | ❌ нет |
| `ProviderInstaller.installIfNeededAsync()` | ❌ нет |
| `GoogleApiAvailability` | ❌ нет |

Приложение не вызывает обновление Security Provider через GMS. Учитывая `minSdkVersion=33` (Android 13), Provider на устройстве уже актуальный (последняя версия – через обновления Google Play System Updates, а не через ProviderInstaller). Однако формально вызов отсутствует.

## вывод

**Тест НЕ ПРОЙДЕН ❌** (L2)

Причина: приложение не вызывает `ProviderInstaller.installIfNeeded()` при старте. Рекомендуется добавить вызов в `MeetWayApp.onCreate()` для гарантии актуальности криптографического провайдера, особенно если приложение использует шифрование AES-GCM и TLS.

**Рекомендация:** добавить `ProviderInstaller.installIfNeeded(this)` в `MeetWayApp.onCreate()` перед инициализацией сетевых соединений.

Уровень теста: L2.
Профиль: MASVS-NETWORK.

---
