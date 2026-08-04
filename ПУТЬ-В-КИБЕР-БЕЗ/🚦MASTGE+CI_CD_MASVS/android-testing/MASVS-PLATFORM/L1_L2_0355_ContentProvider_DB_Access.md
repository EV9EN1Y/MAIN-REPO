# MASTG-TEST-0355: References to Unauthorized Database Access through Content Providers
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0355/

## что проверяет этот тест

Тест MASTG-TEST-0355 (статический) проверяет, не экспортирует ли приложение **ContentProvider** без защиты. ContentProvider - это Android-компонент, который позволяет другим приложениям читать/писать данные приложения через `content://` URI. Если он не защищён permission'ами, любое приложение на устройстве может:

- Прочитать все записи из БД (пользователи, сообщения, токены)
- Вставить/изменить данные
- Узнать структуру таблиц

## какие инструменты использую

беру meetway.apk и смотрю AndroidManifest.xml через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу:
1. Элемент `<provider>` в манифесте
2. Атрибуты `android:exported`, `android:permission`, `android:readPermission`, `android:writePermission`
3. Классы, наследующие `ContentProvider`

## как провожу тест


**шаг 1 - открываю AndroidManifest.xml через jadx-gui и ищу `<provider>`:**
  - Вкладка AndroidManifest.xml → поиск `<provider>` или `ContentProvider`

**шаг 2 - ищу в декомпилированном коде через встроенный поиск jadx:**
  - Text Search (Ctrl+Shift+F) → `ContentProvider`
  - Text Search (Ctrl+Shift+F) → `ContentResolver`
  - Text Search (Ctrl+Shift+F) → `getContentResolver`


## что нашёл



**Результат поиска `<provider>`: ❌ НЕ НАЙДЕНО**

| Что искал | Нашёл? |
|--|--|
| `<provider>` в манифесте | ❌ нет |
| Класс, наследующий ContentProvider | ❌ нет |
| `ContentResolver` или `getContentResolver()` | ❌ нет |

Приложение не использует ContentProvider вообще. Данные хранятся через `SecureStorage` (EncryptedSharedPreferences для JWT), Firebase и локальные файлы через OkHttp

## вывод

**Тест пройден ✅**

Причина: ContentProvider не зарегистрирован – экспортировать и атаковать нечего

Уровень теста: L1, L2
Профиль: MASVS-PLATFORM

---
