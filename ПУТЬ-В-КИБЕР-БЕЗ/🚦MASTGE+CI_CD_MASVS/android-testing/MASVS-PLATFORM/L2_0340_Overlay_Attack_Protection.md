# MASTG-TEST-0340: References to Overlay Attack Protections
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0340/

## что проверяет этот тест

Тест MASTG-TEST-0340 проверяет, использует ли приложение механизмы защиты от **overlay-атак (tapjacking)**. Это когда вредоносное приложение создаёт прозрачное окно поверх вашего – пользователь думает, что нажимает кнопку в легитимном приложении, а на самом деле касается оверлея. Так можно обманом заставить человека подтвердить платёж, дать разрешение или ввести пароль в подставное поле

Android умеет определять, перекрыто ли окно приложения, и отбрасывать касания, но для этого разработчик должен явно включить защиту

- `setFilterTouchesWhenObscured(true)` / `android:filterTouchesWhenObscured="true"`
- `onFilterTouchEventForSecurity()` – кастомная обработка
- `setHideOverlayWindows(true)` + `HIDE_OVERLAY_WINDOWS` permission (API 31+)

## какие инструменты использую

беру meetway.apk и декомпилирую через jadx-gui, потом анализирую код

```bash
jadx-gui ~/Desktop/meetway.apk
```

в jadx ищу
1. Все упоминания `setFilterTouchesWhenObscured`
2. `onFilterTouchEventForSecurity`
3. `setHideOverlayWindows`
4. `FLAG_WINDOW_IS_OBSCURED`
5. permission `HIDE_OVERLAY_WINDOWS` в манифесте
6. targetSdkVersion

## как провожу тест

**шаг 1 – grep по исходному коду**:
```bash
grep -rn "setFilterTouchesWhenObscured\|onFilterTouchEventForSecurity\|FLAG_WINDOW_IS_OBSCURED\|setHideOverlayWindows\|HIDE_OVERLAY_WINDOWS\|filterTouchesWhenObscured" --include="*.kt" --include="*.xml" app/src/main/
```

**шаг 2 – проверяю targetSdkVersion в build.gradle.kts**:
```bash
grep -E "targetSdk|minSdk" app/build.gradle.kts
```

**шаг 3 – проверяю манифест на permission**:
```bash
grep "HIDE_OVERLAY" app/src/main/AndroidManifest.xml
```

**шаг 4 – оцениваю чувствительные экраны** (логин, регистрация, пароли)

## что нашёл

**Результат поиска защитных механизмов: ПУСТО**

| Что искал | Нашёл? |
|--|--|
| `setFilterTouchesWhenObscured` | ❌ нет |
| `onFilterTouchEventForSecurity` | ❌ нет |
| `FLAG_WINDOW_IS_OBSCURED` | ❌ нет |
| `setHideOverlayWindows` | ❌ нет |
| `HIDE_OVERLAY_WINDOWS` permission | ❌ нет |

**targetSdkVersion:** 36 (Android 14+). Для этого API уровень **обязательно** нужно использовать `setHideOverlayWindows(true)`.

**Чувствительные экраны, которые уязвимы:**
- Экран входа: поле Email + поле Пароля + кнопка "Войти"
- Экран регистрации: Имя, Email, Пароль
- Диалог смены пароля в настройках
- Диалог удаления аккаунта (требует пароль)
- Все кнопки в приложении

**Пример атаки:** Злоумышленник ставит приложение с `SYSTEM_ALERT_WINDOW`. Когда пользователь открывает MeetWay и нажимает "Войти", оверлей подменяет нажатие – вместо отправки логина пользователь даёт разрешение на доступ к контактам или подтверждает подписку

## вывод

**Тест НЕ ПРОЙДЕН ❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌❌

Причина: приложение не использует НИ ОДИН механизм защиты от overlay-атак. Ни на одном экране. При этом targetSdk = 36, и для Android 12+ защита (`setHideOverlayWindows`) должна быть обязательной


**Как исправить:**
```kotlin
// В MainActivity.onCreate():
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    window.setHideOverlayWindows(true)
}
```

Добавить в манифест:
```xml
<uses-permission android:name="android.permission.HIDE_OVERLAY_WINDOWS" />
```

---

