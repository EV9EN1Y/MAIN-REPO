# MASTG-TEST-0365: Exported And Unprotected Services That Expose Sensitive Functionality
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0365/

## что проверяет этот тест

Тест MASTG-TEST-0365 проверяет, не экспортирует ли приложение **Service** без защиты. Service с `android:exported="true"` может быть запущен или привязан (bound) из любого приложения. Если Service возвращает данные или выполняет действия - это утечка или несанкционированный доступ

## какие инструменты использую

беру meetway.apk, смотрю манифест:

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу:
1. `<service>` в манифесте
2. `android:exported` у каждого
3. Код сервиса (onStartCommand, onBind, onHandleIntent)

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui и ищу `<service>`:**
  - Вкладка AndroidManifest.xml → поиск `<service>`

**шаг 2 – через встроенный поиск jadx ищу классы сервисов:**
  - Text Search (Ctrl+Shift+F) → `Service` – просматриваю декомпилированные классы

## что нашёл

**Единственный Service в приложении:**
```
MeetWayMessagingService – FCM push-уведомления
```

| Service | exported | Что делает |
|--|--|--|
| `MeetWayMessagingService` | ❌ false | Обрабатывает входящие push от FCM |

Service не экспортирован – запустить его из другого приложения нельзя.

## вывод

**Тест пройден ✅**

Причина: все сервисы с exported=false, запустить извне нельзя.



---
