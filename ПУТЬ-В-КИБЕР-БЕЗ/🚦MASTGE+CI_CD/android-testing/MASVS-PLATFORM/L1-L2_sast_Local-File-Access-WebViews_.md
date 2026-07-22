# MASTG-TEST-0252: References to Local File Access in WebViews
https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0252/



Тест проверяет, может ли злоумышленник через уязвимый WebView получить доступ к локальным файлам приложения и украсть их.

Уязвимость возникает при включении трех настроек:

1. **`setJavaScriptEnabled(true)`** - позволяет выполнять JavaScript.
   
2. **`setAllowFileAccess(true)`** (или не отключено явно) - разрешает WebView загружать локальные файлы.
   
3. **`setAllowFileAccessFromFileURLs(true)`** или **`setAllowUniversalAccessFromFileURLs(true)`** - разрешает JavaScript в локальных файлах читать другие локальные файлы.


Если эти настройки включены вместе, злоумышленник может внедрить вредоносный HTML-файл, который прочитает ваши локальные данные (пароли, ключи, базы данных) и отправит их на свой сервер

----------

## какие инструменты использую

беру meetway.apk и декомпилирую через jadx-gui:

```bash
jadx-gui ~/Desktop/meetway.apk
```

через встроенный поиск jadx ищу:
1. `setJavaScriptEnabled` – включён ли JS в WebView
2. `setAllowFileAccess` – разрешён ли доступ к локальным файлам
3. `setAllowFileAccessFromFileURLs` – может ли JS в локальных файлах читать другие файлы
4. `setAllowUniversalAccessFromFileURLs` – может ли JS из любого источника читать локальные файлы

## как провожу тест

**шаг 1 – открываю meetway.apk в jadx-gui**

**шаг 2 – через встроенный поиск jadx проверяю каждую настройку:**
  - Text Search (Ctrl+Shift+F) → `setJavaScriptEnabled`
  - Text Search (Ctrl+Shift+F) → `setAllowFileAccess`
  - Text Search (Ctrl+Shift+F) → `setAllowFileAccessFromFileURLs`
  - Text Search (Ctrl+Shift+F) → `setAllowUniversalAccessFromFileURLs`

## что нашёл

| Настройка | Статус |
|--|--|
| `setJavaScriptEnabled` | ✅ найден – true (в YandexAuthWebViewActivity) |
| `setAllowFileAccess` | ❌ не найден в коде (по умолчанию true для API < 30) |
| `setAllowFileAccessFromFileURLs` | ❌ не найден (по умолчанию false) |
| `setAllowUniversalAccessFromFileURLs` | ❌ не найден (по умолчанию false) |

Поскольку два метода (`setAllowFileAccessFromFileURLs` и `setAllowUniversalAccessFromFileURLs`) **не используются**, для современных версий Android (API >= 16) их значения по умолчанию – **`false`**. Это означает, что даже если `AllowFileAccess` включён, JavaScript не сможет читать другие локальные файлы.

-------

**Результат:** Тест **MASTG-TEST-0252** пройден

**Обоснование:** В ходе статического анализа кода с помощью jadx было установлено, что в приложении используется WebView (`YandexAuthWebViewActivity`). При этом:

- Метод `setAllowFileAccessFromFileURLs` не используется, следовательно, применяется безопасное значение по умолчанию (`false`)
   
- Метод `setAllowUniversalAccessFromFileURLs` не используется, следовательно, применяется безопасное значение по умолчанию (`false`)
   
- Отсутствие этих настроек делает невозможным чтение локальных файлов из WebView даже при включенном `setJavaScriptEnabled(true)`
   

Таким образом, риск утечки локальных файлов через WebView отсутствует