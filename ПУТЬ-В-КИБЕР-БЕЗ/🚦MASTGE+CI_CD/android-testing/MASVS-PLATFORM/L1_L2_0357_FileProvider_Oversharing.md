# MASTG-TEST-0357: References to Oversharing of File-Based Content Providers
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0357/

## что проверяет этот тест

Тест MASTG-TEST-0357 проверяет, не настроен ли **FileProvider** слишком широко . FileProvider - это специальный ContentProvider для раздачи файлов другим приложениям через `content://` URI. Если в конфигурации `<paths>` указаны слишком   широкие пути (например, `<root-path path="/"/>`), любое приложение может читать любые файлы приложения

## какие инструменты использую

беру meetway.apk и смотрю через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

ищу:
1. `<provider>` с `android:authorities` в манифесте
2. XML-файл с `<paths>` для FileProvider (обычно `res/xml/file_paths.xml`)
3. Вызовы `FileProvider.getUriForFile()` в коде

## как провожу тест

*1 - открываю AndroidManifest.xml через jadx-gui и ищу `<provider>`:**
  - Вкладка AndroidManifest.xml → поиск `FileProvider`, `<provider`

*2 - ищу `getUriForFile` в декомпилированном коде через встроенный поиск jadx:**
  - Text Search (Ctrl+Shift+F) → `getUriForFile`
  - Text Search (Ctrl+Shift+F) → `FileProvider`

*3 - ищу файл конфигурации путей в ресурсах jadx:**
  - Вкладка Resources → `res/xml/` – проверяю наличие `file_paths.xml`, `provider_paths.xml`

## что нашёл

**`FileProvider.getUriForFile()` есть в коде:**
- `SettingScreen.kt` строка 497 – передача файла через URI
- `ChatsScreen.kt` строка 80 – импорт класса

**Но в манифесте `<provider>` НЕТ:** через jadx-gui во вкладке AndroidManifest.xml – ни одного `<provider>` не найдено.

**Файл конфигурации путей: ❌ НЕ НАЙДЕН**

## вывод

**Тест пройден ✅**

Причина: FileProvider не зарегистрирован в манифесте, хотя код использует `getUriForFile()`. Возможно, это ссылка на внешний FileProvider из библиотеки. Поскольку провайдер не объявлен – он не экспортируется и файлы не доступны другим приложениям.



---
