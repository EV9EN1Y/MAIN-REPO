# MASTG-TEST-0356: Runtime Verification of Unauthorized Database Access through Content Providers
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0356/

## что проверяет этот тест

Тест MASTG-TEST-0356 (динамический) - тот же тест, что и 0355, но выполняется на работающем устройстве. Статический анализ может пропустить ContentProvider, если он регистрируется динамически в коде (через `registerContentProvider`). Динамическая проверка пытается выполнить запросы через ADB и смотрит, вернутся ли данные

## какие инструменты использую

Android Debug Bridge (ADB) - телефон OnePlus 8T подключён по USB

```bash
adb shell content query --uri content://com.evgeniy.meetway/
```

## как провожу тест

**шаг 1 - запускаю приложение на телефоне:**
```bash
adb shell monkey -p com.evgeniy.meetway -c android.intent.category.LAUNCHER 1
```

**шаг 2 - выполняю content query по возможным URI:*
```bash
adb shell content query --uri content://com.evgeniy.meetway/
adb shell content query --uri content://com.evgeniy.meetway/data
adb shell content query --uri content://com.evgeniy.meetway/settings
```

## что нашёл

**Все запросы вернули ошибку:**
```
Error while accessing provider:com.evgeniy.meetway
  java.lang.SecurityException: Permission Denial: opening provider
  com.evgeniy.meetway from (null) ... not exported
```

| URI                                      | Результат                               |
| ---------------------------------------- | --------------------------------------- |
| `content://com.evgeniy.meetway/`         | ❌ SecurityException - не экспортирован  |
| `content://com.evgeniy.meetway/data`     | ❌ SecurityEx ception – не экспортирован |
| `content://com.evgeniy.meetway/settings` | ❌ SecurityException – не экспортирован  |

## вывод

**Тест пройден ✅**

Причина: ContentProvider не зарегистрирован (подтверждено runtime). Все запросы blocked

Уровень теста: L1, L2
Профиль: MASVS-PLATFORM

---
