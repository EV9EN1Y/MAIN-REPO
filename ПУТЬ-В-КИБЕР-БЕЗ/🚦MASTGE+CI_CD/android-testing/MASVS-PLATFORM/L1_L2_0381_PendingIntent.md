# MASTG-TEST-0381: References to Insecure PendingIntent Creation
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0381/

## че проверяет этот тест

тест MASTG-TEST-0381 проверяет, безопасно ли созданы **PendingIntent**. PendingIntent – это Intent, который будет выполнен от имени вашего приложения, но в другом процессе/в другое время (например, при тапе по уведомлению). Если PendingIntent неправильно настроен, злоумышленник может

1. **Изменить содержимое Intent** – если нет `FLAG_IMMUTABLE`, получатель может добавить свои extras, изменить action/data
2. **Перехватить Intent** – если Intent неявный (не указан конкретный компонент), другое приложение может зарегистрироваться на тот же action

## какие инструменты использую

беру meetway.apk, декомпилирую через jadx-gui и ищу все вызовы PendingIntent

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу


1. `PendingIntent.getActivity()` – штатный запуск Activity
2. `PendingIntent.getBroadcast()` – отправка broadcast
3. `PendingIntent.getService()` / `getForegroundService()` – запуск сервиса
4. Флаги: `FLAG_IMMUTABLE`, `FLAG_MUTABLE`, `FLAG_UPDATE_CURRENT`
5. Явный/неявный Intent (есть ли `setClass`)

## как провожу тест

**шаг 1 – через встроенный поиск jadx ищу все вызовы PendingIntent:**
  - Text Search (Ctrl+Shift+F) → `PendingIntent`

**шаг 2 – для каждого найденного в jadx проверяю:**
  - Есть ли `FLAG_IMMUTABLE`
  - Явный ли Intent (указан класс через второй параметр)

**шаг 3 – проверяю minSdkVersion в манифесте через jadx (вкладка AndroidManifest.xml)**

## что нашёл

**Все PendingIntent в MeetWay (4 штуки):**

| Где | Метод | Intent (явный?) | Флаги | Статус |
|--|--|--|--|--|
| `showNotification()` | `getActivity()` | ✅ `MainActivity::class.java` | ✅ `FLAG_IMMUTABLE` | ✅ |
| `showNotification()` | `getBroadcast()` | ✅ `NotificationActionReceiver` | ✅ `FLAG_IMMUTABLE` | ✅ |
| `showNotification()` | `getBroadcast()` | ✅ `NotificationActionReceiver` | ✅ `FLAG_IMMUTABLE` | ✅ |
| `showLocalNotification()` | `getActivity()` | ✅ `MainActivity::class.java` | ✅ `FLAG_IMMUTABLE` | ✅ |

Все 4 PendingIntent:
- Используют явный Intent (конкретный класс указан)
- Имеют `FLAG_IMMUTABLE` (intent нельзя изменить)
- minSdk = 33 (FLAG_IMMUTABLE обязателен, и он есть)

## вывод

**Тест пройден ✅**

Причина: все PendingIntent созданы с FLAG_IMMUTABLE и с явным Intent. Перехватить или изменить их нельзя.


---
