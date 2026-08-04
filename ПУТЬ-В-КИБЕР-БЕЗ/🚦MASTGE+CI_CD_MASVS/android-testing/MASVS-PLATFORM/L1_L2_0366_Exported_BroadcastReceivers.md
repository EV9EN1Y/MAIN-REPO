# MASTG-TEST-0366: Exported And Unprotected Broadcast Receivers That Expose Sensitive Functionality
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0366/

## что проверяет этот тест

Тест MASTG-TEST-0366 проверяет, не экспортирует ли приложение **BroadcastReceiver** без защиты. Receiver с `android:exported="true"` может получать широковещательные интенты от любого приложения. Если `onReceive()` выполняет чувствительные действия (отправка сообщений, изменение данных) – злоумышленник может вызвать это, отправив поддельный broadcast

## какие инструменты использую

беру meetway.apk, смотрю манифест + код:

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу:
1. `<receiver>` в манифесте – exported и permission
2. Context-registered receivers в коде (`registerReceiver`)
3. Методы `sendBroadcast()`

## как провожу тест

**шаг 1 – открываю AndroidManifest.xml через jadx-gui и ищу `<receiver>`:**
  - Вкладка AndroidManifest.xml → поиск `<receiver>`

**шаг 2 – через встроенный поиск jadx ищу все broadcast-related вызовы:**
  - Text Search (Ctrl+Shift+F) → `BroadcastReceiver`
  - Text Search (Ctrl+Shift+F) → `registerReceiver`
  - Text Search (Ctrl+Shift+F) → `sendBroadcast`

## что нашёл

**Receiver в манифесте:**

| Receiver | exported | Что делает |
|--|--|--|
| `NotificationActionReceiver` | ❌ false | REPLY_ACTION / MARK_AS_READ из уведомлений |

**Context-registered receivers (внутренние broadcast-ы):**
- `com.evgeniy.meetway.APP_ACTIVE` – уведомление ViewModel при активации
- `com.evgeniy.meetway.OPEN_CHAT` – навигация на чат
- `com.evgeniy.meetway.NAVIGATE_TO_WELCOME` – редирект на приветственный экран

Все эти broadcast-ы отправляются и принимаются **внутри одного приложения**, не экспортированы.

## вывод

**Тест пройден ✅**

Причина: все receivers exported=false, context-registered receivers используются только внутри прилы

---
