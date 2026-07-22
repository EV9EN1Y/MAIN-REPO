# MASTG-TEST-0258: References to Keyboard Caching Attributes in UI Elements
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0258/

тест проверяет, не кэширует ли клавиатура то, что ты вводишь в чувствительные поля (пароли, номера карт, личные данные). Если поле не помечено специальным флагом, клавиатура может сохранять введённый текст, чтобы предлагать его потом в других приложениях - а это уже утечка

В Android есть специальные атрибуты для полей ввода, которые говорят системе: "то, что здесь вводят, не нужно запоминать и предлагать снова". Это делается через:

- В XML-разметке: `android:inputType` с флагами `textNoSuggestions`, `textVisiblePassword`, `textWebPassword` и т.д
   
- В коде (или Jetpack Compose): `setInputType(...)` или `KeyboardOptions` с отключённой автокоррекцией
   

**Тест считается проваленным**, если в приложении есть хоть одно поле для ввода чувствительных данных (пароль, email, телефон, паспортные данные), у которого **не выставлен** соответствующий защитный флаг. То есть клавиатура может запомнить и предложить эти данные в будущем


---------

чек лист:

нужно найти в коде все текст поля и проверить их настройку!

```q
|Флаг|Защита от кэширования|

|`textNoSuggestions`|✅ Отключает подсказки и кэширование|
|`textVisiblePassword`|✅ Скрывает вводимый текст и отключает кэширование|
|`textWebPassword`|✅ Аналог `textVisiblePassword` для веб-форм|
|`textWebEmailAddress`|✅ Отключает кэширование для email (обычно безопасно)|
|`phone`|⚠️ Для номеров телефонов – обычно не кэшируется, но лучше проверить|

**Опасные (незащищённые) флаги для чувствительных данных:**

|Флаг|Риск|

|`text` (обычный текст)|❌ Кэшируется клавиатурой|
|`textAutoCorrect`|❌ Включает автокоррекцию и кэширование|
|`textCapSentences` / `textCapWords`|❌ Может кэшироваться|
|`textEmailAddress` (без `textNoSuggestions`)|❌ Может кэшироваться, если нет доп. флагов|
```

------

##  запускаю  jadx-gui + использую LLM OPEN CLAW для помощи в поиске
```c
jadx-gui ~/Desktop/meetway.apk
```


## Результаты теста 

**Приложение:** MeetWay Android (Kotlin + Jetpack Compose)
**Метод:** SAST - поиск всех `OutlinedTextField` в исходном коде, проверка `KeyboardOptions` и `visualTransformation`


### Общее количество текстовых полей

Найдено ~55 `OutlinedTextField` в 14 файлах. Большинство - поля для нечувствительных данных (поиск, названия стран/городов, описания, комментарии)

### Критические проблемы

#### 1. RegistrationScreen.kt - 3 поля без KeyboardOptions

| Поле | Строка | KeyboardOptions | visualTransformation | Статус |
|--|--|--|--|--|
| **Имя** | 108 | ❌ Отсутствует | ❌ Нет | **FAIL** |
| **Никнейм** | 127 | ❌ Отсутствует | ❌ Нет | ⚠️ LOW |
| **Email** | 146 | ❌ Отсутствует | ❌ Нет | **FAIL** |
| **Пароль** | 165 | ❌ `KeyboardType.Password` не указан | ✅ `PasswordVisualTransformation()` | **FAIL** |

#### 2. SettingScreen.kt - 4 поля с паролями без KeyboardOptions

| Поле | Строка | KeyboardOptions | Статус |
|--|--|--|--|
| Текущий пароль | 588 | ❌ Отсутствует | **FAIL** |
| Новый пароль | 597 | ❌ Отсутствует | **FAIL** |
| Подтверждение | 606 | ❌ Отсутствует | **FAIL** |
| Пароль для удаления | 731 | ❌ Отсутствует | **FAIL** |

У всех этих полей стоит `visualTransformation = PasswordVisualTransformation()`, но нет `KeyboardOptions(keyboardType = KeyboardType.Password)`. `PasswordVisualTransformation` даёт **только визуальное скрытие** символов и **не отключает** кэширование клавиатуры ☠️❌❌❌

#### 3. EnterAccountScreen.kt - все ок

| Поле | Строка | KeyboardOptions | Статус |
|--|--|--|--|
| Email | 200 | ✅ `KeyboardType.Email` | **PASS** |
| Пароль | 221 | ✅ `KeyboardType.Password` | **PASS** |

#### 4. RecoverPasswordScreen.kt - все ок

| Поле | Строка | KeyboardOptions | Статус |
|--|--|--|--|
| Email | 100 | ✅ `KeyboardType.Email` | **PASS** |

### Итог: тест провален

**8 полей провалили тест** - все на экранах регистрации и настроек:

- RegistrationScreen: Имя, Email, Пароль (3 FAIL)
- SettingScreen: Текущий пароль, Новый пароль, Подтверждение, Пароль для удаления (4 FAIL)
- RegistrationScreen Никнейм (1 LOW - нечувствительные данные, но лучше поправить)

### Критичность: Высокая

Пароли и email вводятся на экранах аутентификации и смены пароля. Если клавиатура (Gboard или сторонняя) кэширует эти данные, они могут быть доступны через механизм автозаполнения другому приложению





--------
### Как исправить

**Для полей с паролем:**
```kotlin
keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password, imeAction = ImeAction.Done)
```

**Для email в RegistrationScreen:**
```kotlin
keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email, autoCorrect = false)
```

**Для имени в RegistrationScreen:**
```kotlin
keyboardOptions = KeyboardOptions(autoCorrect = false)
```

### Конкретные строки для исправления

**RegistrationScreen.kt:**
- Строка 108 (Имя): добавить `KeyboardOptions(autoCorrect = false)`
- Строка 146 (Email): добавить `KeyboardOptions(keyboardType = KeyboardType.Email)`
- Строка 165 (Пароль): заменить на `KeyboardOptions(keyboardType = KeyboardType.Password, imeAction = ImeAction.Done)` + сохранить `PasswordVisualTransformation()`

**SettingScreen.kt:**
- Строки 588, 597, 606, 731: добавить `keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)`

---


