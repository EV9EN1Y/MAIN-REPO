# MASTG-TEST-0291: References to Screen Capturing Prevention APIs
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0291/

этот тест проверяет, использует ли приложение **`FLAG_SECURE`** для защиты чувствительных экранов от скриншотов, записи экрана и отображения в Recents (список недавних приложений)

Без `FLAG_SECURE`:
- Любое приложение на устройстве может сделать скриншот экрана
- Злоумышленник через ADB может захватить скриншот или запись экрана (`screenrecord`)
- При сворачивании приложения его содержимое видно в списке Recents

Задача теста - проверить, что на всех экранах, где вводятся пароли, ключи или другие чувствительные данные, установлен `WindowManager.LayoutParams.FLAG_SECURE`

------

проверить это статически - достаточно найти упоминания `FLAG_SECURE` в коде
либо проверить динамически - через скриншот экрана с паролем/чувствительными данными (он должен быть чёрным или пустым)

----

для защиты используется:
**`FLAG_SECURE`** -флаг окна, который запрещает системе делать скриншоты и запись экрана

в коде компоуз это выглядит так:
```kotlin
// Вариант 1: через window в Activity
window.setFlags(
    WindowManager.LayoutParams.FLAG_SECURE,
    WindowManager.LayoutParams.FLAG_SECURE
)

// Вариант 2: через Activity.setRequestedOrientation + FLAG_SECURE
// Вариант 3: в Jetpack Compose через SideEffect
SideEffect {
    activity.window.setFlags(
        WindowManager.LayoutParams.FLAG_SECURE,
        WindowManager.LayoutParams.FLAG_SECURE
    )
}
```

для проверки нужно искать:
- Вызовы `setFlags` с `FLAG_SECURE` в классах `Activity`
- Вызовы `window.addFlags(FLAG_SECURE)`
- `SideEffect { activity.window.setFlags(FLAG_SECURE, FLAG_SECURE) }` в Compose

-------
##  запускаю  jadx-gui
```c
jadx-gui ~/Desktop/meetway.apk
```

## результаты теста (SAST) 

**Метод:** поиск `FLAG_SECURE` во всех `.kt` и `.xml` файлах MeetWay Android

**Результат поиска:** ❌ `FLAG_SECURE` НЕ НАЙДЕН нигде в коде

| Что проверяли | Найдено |
|--|--|
| `FLAG_SECURE` в Activity | ❌ Нет |
| `window.addFlags(FLAG_SECURE)` | ❌ Нет |
| `window.setFlags(FLAG_SECURE)` | ❌ Нет |
| `SideEffect` с FLAG_SECURE в Compose | ❌ Нет |
| `onPause()`/`onResume()` с подменой окна | ❌ Нет |

**Подробности:**

Все три Activity (`MainActivity`, `YandexAuthWebViewActivity`, `AuthRedirectActivity`) не устанавливают `FLAG_SECURE` ни в `onCreate()`, ни в Compose-контенте

`MainActivity` содержит lifecycle-коллбеки (`onPause`, `onStop`, `onResume`), но ни один из них не меняет флаги окна для скрытия содержимого

**Чувствительные экраны без защиты:**
- Экран входа (EnterAccountScreen) -оле ввода пароля ❌
- Экран регистрации (RegistrationScreen) - поле ввода пароля ❌
- Диалог смены пароля (SettingScreen) ❌
- Диалог удаления аккаунта (SettingScreen) ❌
- Экран чата (ChatsScreen) - переписка ❌
- Экран восстановления пароля (RecoverPasswordScreen) ❌



### **итог:❌❌❌❌❌❌  тест провален ❌❌❌**



Приложение **не использует `FLAG_SECURE`** ни на одном экране. Это означает:
1. Любой скриншот (через кнопки громкости) может захватить пароль/переписку
2. Запись экрана через `screenrecord` или любую софтину не блокируется
3. Содержимое приложения видно в Recents при сворачивании
4. Все диалоги с паролями также не защищены

**Критичность: Высокая** - пароли и личные сообщения могут быть захвачены через скриншот, запись экрана или Recents

**Как исправить:**

Добавить `FLAG_SECURE` в `MainActivity.onCreate()`:
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    window.setFlags(
        WindowManager.LayoutParams.FLAG_SECURE,
        WindowManager.LayoutParams.FLAG_SECURE
    )
    enableEdgeToEdge()
    // ... остальной код
}
```

ИЛИ на уровне конкретных компоуз-экранов с чувствительными данными через `SideEffect`:
```kotlin
@Composable
fun SensitiveScreen(activity: Activity = LocalContext.current as Activity) {
    SideEffect {
        activity.window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    }
    // ... UI
}
```

`FLAG_SECURE` блокирует ВСЕ скриншоты и запись экрана для приложения - это минус для UX/отладки, но плюс для безопасности-

---


