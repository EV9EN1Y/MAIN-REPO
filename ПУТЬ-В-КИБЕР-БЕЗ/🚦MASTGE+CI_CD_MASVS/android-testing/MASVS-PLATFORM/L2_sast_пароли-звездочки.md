# MASTG-TEST-0316: App Exposing User Authentication Data in Text Input Fields
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0316/

 тест проверяет, не отображается ли вводимый пароль, PIN-код или OTP-код **открытым текстом** в поле ввода. Иными словами - скрыты ли символы точками (или звёздочками) во время ввода, чтобы никто не мог подсмотреть их через плечо

--------

### чек-лист для  JADX

1. **откр прилу и найти все экраны с полями ввода:** в JADX открыть папку `resources/res/layout/` и проверить все XML-файлы
   
2. **Найти поля, которые могут быть для пароля/кода:**
   
    - искать ID с названиями `password`, `pin`, `code`, `otp`, `pass`, `pwd`
     
    - искать подсказки (атрибут `android:hint`) с текстом "Пароль", "Код", "PIN"
      
3. **Проверить их `inputType`:**
   
    - **Безопасно:** `textPassword`, `textWebPassword`, `numberPassword`, `textVisiblePassword`
     
    - **Небезопасно:** `text`, `textCapWords`, `number` (если для PIN) и т.п.
     
4. **Проверить код на программную установку:** ищи вызовы `setInputType()` для этих полей и убедись, что они не меняют защиту на опасную (`InputType.TYPE_CLASS_TEXT` и т.п.)
   
5. **Проверить Jetpack Compose:** если видишь импорты `androidx.compose.material.TextField` и рядом `SecureTextField` - убедись, что `TextObfuscationMode` не установлен в `Visible`

---------

## Результаты теста (SAST) 

**Метод:** поиск всех `OutlinedTextField` и проверка наличия `PasswordVisualTransformation()` / `VisualTransformation.None` в параметрах. Поиск полей с `pin`, `otp`, `code`, `pwd` в ID и hint

**Приложение:** MeetWay Android (Kotlin + Jetpack Compose) – без XML-разметки, все поля через `OutlinedTextField` Compose

**Найденные поля с паролями:**

| Экран | Поле | Строка | `PasswordVisualTransformation` | Кнопка "Показать" | Статус |
|--|--|--|--|--|--|
| `EnterAccountScreen.kt` | Пароль (вход) | 228 | ✅ `PasswordVisualTransformation()` | ✅ Есть (`showPassword` toggle) | **PASS** |
| `RegistrationScreen.kt` | Пароль (регистрация) | 170 | ✅ `PasswordVisualTransformation()` | ✅ Есть | **PASS** |
| `SettingScreen.kt` | Текущий пароль | 592 | ✅ `PasswordVisualTransformation()` | ✅ Есть (общий toggle) | **PASS** |
| `SettingScreen.kt` | Новый пароль | 601 | ✅ `PasswordVisualTransformation()` | ✅ Есть | **PASS** |
| `SettingScreen.kt` | Подтверждение пароля | 610 | ✅ `PasswordVisualTransformation()` | ✅ Есть | **PASS** |
| `SettingScreen.kt` | Пароль для удаления | 738 | ✅ `PasswordVisualTransformation()` | ❌ Нет (всегда скрыт) | **PASS** |

**Другие чувствительные поля (PIN/OTP/коды):**

- PIN-коды: ❌ Не найдены (в приложении нет PIN-входа)
- OTP-коды: ❌ Не найдены (нет двухфакторной аутентификации)
- Коды подтверждения: ❌ Не найдены

**Непарольные поля (не требуют скрытия):**

| Экран                      | Поле    | Строка | Почему ок                           |
| -------------------------- | ------- | ------ | ----------------------------------- |
| `EnterAccountScreen.kt`    | Email   | 200    | Открытый текст - ожидаемо для email |
| `RegistrationScreen.kt`    | Имя     | 108    | Открытый текст - имя не секрет      |
| `RegistrationScreen.kt`    | Никнейм | 127    | Открытый текст - никнейм            |
| `RegistrationScreen.kt`    | Email   | 146    | Открытый текст - ожидаемо           |
| `RecoverPasswordScreen.kt` | Email   | 100    | Открытый текст - ожидаемо           |

**Механизм скрытия:**

Все поля с паролями используют `PasswordVisualTransformation()`:
```kotlin
visualTransformation = if (showPassword) VisualTransformation.None else PasswordVisualTransformation()
```

Кнопка "Показать пароль" (`showPassword` toggle) даёт пользователю возможность временно увидеть вводимые символы - это штатный UX-паттерн, который не считается нарушением.

**Дополнительно - checkBox "Показать пароль" в SettingScreen:**
```kotlin
Checkbox(checked = showPassword, onCheckedChange = { showPassword = it })
Text("Показать пароль")
```
Управляет всеми тремя полями смены пароля одновременно

**Поле для удаления аккаунта:**
```kotlin
visualTransformation = PasswordVisualTransformation()
```
Всегда скрыто - кнопки "Показать" нет. Это осознанное решение для дополнительной безопасности при удалении аккаунта

## **Итог: тест пройден

Все 6 полей с паролями корректно скрывают вводимые символы через `PasswordVisualTransformation()`. PIN/OTP/коды в приложении отсутствуют. Непарольные поля не требуют скрытия

---




