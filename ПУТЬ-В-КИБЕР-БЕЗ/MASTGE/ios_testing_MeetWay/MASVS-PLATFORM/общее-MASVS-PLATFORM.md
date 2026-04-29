**MASVS-PLATFORM** (платформа iOS). Это тесты на **безопасность взаимодействия приложения с iOS-средой** - как приложение общается с другими приложениями, с системой, с пользователем

-----

данный список - можно использовать как чек-лист

--------

## 📱 IPC МЕХАНИЗМЫ (межпроцессное взаимодействие)

### MASTG-TEST-0056: Sensitive Data Exposed via IPC

**О чём:** Проверяет, не передаёт ли приложение чувствительные данные (токены, пароли, личную информацию) через механизмы общения между приложениями.

**Как тестировать:** Отслеживать отправку данных через `UIPasteboard`, `Custom URL Schemes`, `App Extensions`, `Shared UserDefaults`.

через радар 2 - искать вызовы и тоже самое через frida смотреть динамически!
```
generalPasteboard
pasteboardWithName
 setItems:options
canOpenURL
application:openURL:options
NSExtension  в Info.plist
initWithSuiteName:
suiteName
```

**Риск:** Другое приложение-шпион может перехватить данные

---

##  UI И СКРИНШОТЫ

### MASTG-TEST-0057: Sensitive Data Disclosed Through User Interface

**О чём:** Проверяет, не показывается ли на экране слишком много чувствительной информации (пароли, номера карт, персональные данные), которые может подсмотреть посторонний. 
АЙФОН всегда делает скрин экрана - когда приложение убираем в бекграунд, и в режиме многозадачности видно эти свернутые экраны

**Как тестировать:** Оценить все экраны приложения глазами пентестера, и где нужно - принять меры - чтобы прятать эти значения, блюрить например..

### MASTG-TEST-0059: Auto-Generated Screenshots for Sensitive Information

**О чём:** iOS делает скриншот приложения перед уходом в фон (в switcher). Если на этом скриншоте видны личные данные - утечка.

**Защита:** все просто и банально - затемнять экран или скрывать чувствительные данные в `applicationDidEnterBackground`

### MASTG-TEST-0290: Screenshots During App Backgrounding

**О чём:** То же самое, что и 0059, проверить - блюриться или подменяется ли экран при срабатывании метода applicationDidEnterBackground

---

##  ПРАВА И РАЗРЕШЕНИЯ

### MASTG-TEST-0069: Testing App Permissions

**О чём:** Проверяет, какие разрешения запрашивает приложение (камера, контакты, геолокация, фото,蓝牙, уведомления) и обоснованно ли их использует

**Как тестировать:** Смотреть `Info.plist` (`NS*UsageDescription`) и реальные запросы на устройстве

3 этапа проверок!
1 руками тыкать приложение
2 sast  смотреть что есть в коде и Info
3 dast  - frida - в динамике смотреть че вызывается

нужно - просто тестировать руками 
но также нужно проверить Info.plist
Проверить файл `.entitlements`
искать api вызовы в коде - где вызывается то или иное разрешение
можно использовать сканеры CLI-инструменты - на поиск небезопасных конфигураций и разрешений

и через frida можно следить трейсы  `frida-trace`
и проверить, че там есть вообще в вызовах (заранее пейлоад подготовить и через него фильровать, иначе логов будет триллионы)



---

##  ВЗАИМОДЕЙСТВИЕ С ДРУГИМИ ПРИЛОЖЕНИЯМИ

### MASTG-TEST-0070: Testing Universal Links

**О чём:** Проверяет, правильно ли приложение обрабатывает универсальные ссылки (например, `myApp.com/profile` открывает приложение). Нельзя ли подменить ссылку или перехватить данные

Universal Links - это механизм, который позволяет приложению «перехватывать» определённые HTTPS-ссылки и открывать их внутри себя, а не в браузере

как тестить:
внутри entitlements нужно смотреть ключ  com.apple.developer.associated-domains и там есть applinks - это домены которые приложение считает «своими»
(нужно подробнее читать про этот тест)

### MASTG-TEST-0071: Testing UIActivity Sharing

**О чём:** Проверяет, не отправляет ли приложение через системное меню "Поделиться" (`UIActivityViewController`) чувствительные данные (документы, фотографии, текст)

можно тестить и динамически и статически

через фрида перехватывать события при нажатии на кнопки  "Поделиться" (Share) и смотреть, че там происходит, там не должно быть конфидентц данных

```
# Найти вызовы UIActivityViewController
[0x00000000]> izz~UIActivityViewController
[0x00000000]> izz~activityItems
```
### MASTG-TEST-0072: Testing App Extensions

**О чём:** Проверяет расширения приложения (например, клавиатура, виджет, шэринг). Не утекают ли через них данные или нет ли у них избыточных прав

статически искать расширения
```
# В Info.plif приложения (не расширения)
cat Info.plist | grep -A5 "NSExtension"

# Или найти .appex файлы в IPA
unzip MeetWay.ipa
find Payload -name "*.appex"

проверять права
# Извлечь entitlements расширения
codesign -d --entitlements :- Payload/MeetWay.app/PlugIns/Keyboard.appex
```

динамически:
```
# Найти PID расширения
frida-ps -Uai | grep -i "keyboard\|share\|widget"
# Подключиться и смотреть логи происходящего там, лишнего не должно быть,  вдруг там данные уходят куда-то...
frida -U <PID> -e 'console.log("[*] Watching extension...");'
```

### MASTG-TEST-0075: Testing Custom URL Schemes

**О чём:** Проверяет кастомные схемы типа `meetway://`. Нельзя ли вызвать опасные действия через URL (например, перевести деньги, удалить данные).

**Риск:** Другое приложение может вызвать вашу схему

**Custom URL Scheme** - это **собственный протокол приложения** (типа `mybank://`), который может вызвать **любое другое приложение или сайт**, открыв такую ссылку. Если нет защиты — злоумышленник может заставить приложение удалить данные, отправить деньги или открыть опасный экран без ведома пользователя

то есть это такие схемы - по которым нажав где-либо в телефоне на ссылку - откроется приложение - и в самом приложении например , откроется нужны чат!

то есть такая ссылка может выглядеть не стандартно `https://` 
а вот так кастомно `monobank://`  и ios , при начатии на такую ссылку , сперва проверит, если ли на внутри info.plist какого-либо приложения на телефоне такой ключ - что читает monobank://, и тогда ios откроет данное приложение - а далее , те параметры - что есть в ссылке - могут уже выполнять - какую-либо функцию внутри приложения!

и вот нужно проверить - нет ли опасных обрабочиков этих параметров , которые выполняли ли бы опасные действия!

как это дело искать:
sast
```
# В Info.plist
cat Info.plist | grep -A5 "CFBundleURLSchemes"

# Или через radare2 в бинарнике
[0x00000000]> izz~CFBundleURLSchemes

и потом через например браузер открывать все комбинации ссылок (параметров) что я нашел 
myapp://test
и смотреть реакцию
```

dast 
```
frida

можно запускать эти процессы
xcrun simctl openurl booted "myapp://delete-all-data"

или перехватывать входящие url - переходя по ссылкам этим кастомным
```

1 - не должно быть опасных действий
2 - на важные действия - должно быть предупреждение в приложении
3 - должна быть проверка источника url
4- проверять, можно ли подменять данные 
5 - ну и там не должно быть паролей и токенов

---

## 📋 UIPASTEBOARD (БУФЕР ОБМЕНА)

### MASTG-TEST-0073: Testing UIPasteboard

Проверить, что приложение не кладёт в **общий** буфер обмена (который видят все приложения) чувствительные данные

в ios есть два буфера - общий и частный (частный - для каждого приложения локально)

**Риск:** Если ты кладёшь пароль в общий буфер - любое приложение-шпион (фонарик, калькулятор, игра) может его прочитать

тесты

sast
```c
# Найти использование UIPasteboard
[0x00000000]> izz~UIPasteboard
[0x00000000]> izz~generalPasteboard
[0x00000000]> izz~setString
[0x00000000]> izz~setItems
```

dast frida   - в динамике следить - что и куда записывается
```c
// hook_general_pasteboard.js
var UIPasteboard = ObjC.classes.UIPasteboard;
var generalPB = UIPasteboard.generalPasteboard();
// Следить за записью
Interceptor.attach(generalPB.setString, {
    onEnter: function(args) {
        var str = ObjC.Object(args[2]);
        console.log("[⚠️] GENERAL PASTE: " + str);
    }
});
```

если общий - то там не должно быть чувствительных данных

вот например в приложении банка Tmobile - можно скопировать номер карты и другие данные - и вставить в любом месте айфона (то есть - они в общем буфере) при этом - есть риск, что какое -то приложение - может спокойно брать эти данные и делать с ними что захочет на законных основаниях, так как это общий буфер

### MASTG-TEST-0276: Use of iOS General Pasteboard

**О чём:** Проверяет, использует ли приложение **общий** буфер обмена (доступен всем приложениям)
типо тоже - как и предыдущий тест, но

тестить также
```q
# через radare2
[0x00000000]> izz~generalPasteboard

# или  strings
strings MeetWay.app | grep -i "generalPasteboard"
```

ну и динамически = копировать чувствит данные в одном месте и вставлять в другое


### MASTG-TEST-0277: Sensitive Data in General Pasteboard at Runtime

**О чём:** Проверяет, кладёт ли приложение чувствительные данные (пароль, токен) в общий буфер во время работы

все тоже самое - но типо в динамике смотреть
```
вручную
    Открыть приложение
Выполнить разные действия (логин, копирование данных, шаринг)
После каждого действия - переключиться в другое приложение и попробовать вставить
Если вставился пароль, токен, номер карты → тест не пройден
```

или через фрида перехватывать запись  в буфер
```q
// track_general_pasteboard.js
var UIPasteboard = ObjC.classes.UIPasteboard;
Interceptor.attach(UIPasteboard["- setString:"].implementation, {
    onEnter: function(args) {
        var str = ObjC.Object(args[2]);
        console.log("\n[⚠️] WRITE to general pasteboard: " + str);
        
        // Эвристика: ищем паттерны паролей/токенов
        if (str.toString().length > 20 && !str.toString().includes(" ")) {
            console.log("[🔴] POSSIBLE TOKEN/PASSWORD DETECTED!");
        }
    }
});

Interceptor.attach(UIPasteboard["- setItems:"].implementation, {
    onEnter: function(args) {
        var items = ObjC.Object(args[2]);
        console.log("\n[⚠️] WRITE items to general pasteboard: " + items);
    }
});

frida -U MeetWay -l track_general_pasteboard.js
```

### MASTG-TEST-0278: Pasteboard Contents Not Cleared After Use

проверка на то, сколько времени хранятся данные и еще что происходит с ними - после вставки, удаляются или нет

**О чём:** Проверяет, очищает ли приложение буфер обмена после того, как вставил данные
риск понятен, если данные бесконечно храняться - то шансы потерять данные увеличиваются, посетил сайт - буфер ушел... здатути!

🧐 можно тестить вручную - засечь время - через которое данные стираются

либо запустить фриду - и пусть она постоянно проверяет - че там происходит с фуфером

```q
// track_pasteboard_clear.js
var UIPasteboard = ObjC.classes.UIPasteboard;
var generalPB = UIPasteboard.generalPasteboard();

// Отслеживаем установку пустой строки
Interceptor.attach(generalPB.setString.implementation, {
    onEnter: function(args) {
        var str = ObjC.Object(args[2]);
        if (str.toString() === "") {
            console.log("[🧹] Pasteboard CLEARED (empty string set)");
        } else {
            console.log("[✏️] Pasteboard WRITE: " + str);
        }
    }
});

// Альтернатива: проверяем состояние каждые 2 секунды
setInterval(function() {
    var current = generalPB.string();
    if (current && current.toString().length > 0) {
        console.log("[⚠️] Pasteboard still contains: " + current);
    }
}, 2000);
```


либо искать статически
```
[0x00000000]> izz~UIPasteboard
[0x00000000]> izz~generalPasteboard

[0x00000000]> izz~dispatch_after
[0x00000000]> izz~_dispatch_after
[0x00000000]> sym.imp.dispatch_after

[0x00000000]> izz~setString
[0x00000000]> izz~setItems


и все че найдется - понять что это буфер или нет, и то что буфер -
проверять где вызывается
axt @0x00xxxxxx

искать таймеры
[0x00000000]> izz~NSTimer
[0x00000000]> izz~scheduledTimerWithTimeInterval
```


   Copy
- Paste
- Открыть другое приложение
- Попробовать Paste again
- Если вставилось → **fail**
### MASTG-TEST-0279: Pasteboard Contents Not Expiring

**О чём:** Проверяет, не хранятся ли данные в буфере бесконечно долго без таймаута

это о том, что данные вообще никогда из буфера не уходят

   Copy
- Подождать 2–5 минут (ничего не вставлять)
- Открыть другое приложение
- Попробовать Paste
- Если вставилось → **fail**

### MASTG-TEST-0280: Pasteboard Contents Not Restricted to Local Device

**О чём:** Проверяет, синхронизируется ли буфер обмена через iCloud между устройствами. Если да - то секреты могут улететь на другой телефон

## Риск:

Если буфер синхронизируется:
1. Пользователь скопировал пароль на iPhone
2. Пароль автоматически улетел в iCloud
3. На Mac'e (где пользователь забыл выйти из iCloud) любое приложение может прочитать этот пароль из буфера

можно тестить - через фриду следить за записью в   iCloud есть она или нет
```js
// track_icloud_pasteboard.js
var NSUserDefaults = ObjC.classes.NSUserDefaults;
var ubiquityStore = NSUserDefaults.standardUserDefaults();

// Если есть вызовы синхронизации с iCloud
Interceptor.attach(ubiquityStore.synchronize.implementation, {
    onLeave: function() {
        console.log("[☁️] iCloud sync triggered!");
    }
});
```

через радар2
```
 - ищем использование iCloud синхронизации
   
[0x00000000]> izz~NSUbiquitousKeyValueStore
[0x00000000]> izz~iCloud
[0x00000000]> izz~NSUbiquitous
```

вручную

- На iPhone: **Настройки → **Основные** → **AirPlay   **Handoff  (непрерывность)**** → iCloud Drive**
- Посмотреть, включён ли **Universal Clipboard**
- Отключить его (для теста)
- Скопировать данные в приложении
- На Mac (под тем же Apple ID) нажать «Вставить»
- **Если вставилось** → синхронизация есть ❌

также вот тут должна быть настройка

**«Настройки» → «Конфиденциальность и безопасность» → «Буфер обмена»** - там должен быть включен общий переключатель «Общий доступ к буферу обмена между устройствами»

и вот если AirPlay включен и Handoff - то скопировал на айфоне - и на маке - можно вставить эти данные... (условие - если на устройствах общий apple id)

я постоянно пользуюсь этой функцией
но если кто-то паралельно со мной будет это использовать - то может получить мои данные - при ЭПИЧЕСКОМ СТЕЧЕНИИ обстоятельств


---



## 🌐 WEBVIEW

### MASTG-TEST-0076: Testing iOS WebViews

**Общий тест** на безопасность WebView

**Суть:** Проверить, что WebView (встроенный браузер внутри приложения) не имеет дыр, через которые злоумышленник может украсть данные или выполнить вредоносный код

Это окно внутри приложения, где открываются веб-страницы. Например:

- Экран авторизации через соцсети
- Показ баннеров и статей
- Формы оплаты внутри приложения

**Риск:** Если WebView настроен неправильно, злоумышленник может вставить свой JavaScript или открыть локальные файлы приложения

протестить 
SAST
вот так устаревший UIWebView небезопасный
новый WKWebView хороший!

радар2
```
[0x00000000]> izz~UIWebView
[0x00000000]> izz~WKWebView

также найти настройки
# SAST: найти настройки WKWebView (смотрим - подрублен ли JS)
[0x00000000]> izz~preferences.javaScriptEnabled

Если JavaScript включён → риск XSS (нужны дополнительные проверки)

```

также нужно проверить отключены ли опасные возможности
```
[0x00000000]> izz~allowFileAccessFromFileURLs
[0x00000000]> izz~allowUniversalAccessFromFileURLs
[0x00000000]> javaScriptCanOpenWindowsAutomatically

(доступ к файлам)
доступ к любому контенту
открытие окон без спроса
```

Можно ли через WebView прочитать `UserDefaults`, `sqlite` базы, файлы с токенами
```
# SAST: ищем загрузку HTML из папок приложения
[0x00000000]> izz~loadFileURL
[0x00000000]> izz~loadHTMLString
```

в динамике
следить за загрузкой URL
```js
// hook_webview.js
var WKWebView = ObjC.classes.WKWebView;

Interceptor.attach(WKWebView["- loadRequest:"].implementation, {
    onEnter: function(args) {
        var request = ObjC.Object(args[2]);
        var url = request.URL().absoluteString();
        console.log("[🌐] WebView loading: " + url);
        
        if (url.includes("file://")) {
            console.log("[🔴] CRITICAL: Local file access detected!");
        }
    }
});
```

внедрить свой JavaScript (проверить XSS)
```js
// В консоли Frida после аттача к процессу
var webView = ObjC.classes.WKWebView.alloc().init();
webView.evaluateJavaScript_completionHandler_("alert('XSS')", null);
```

```js
// Попробовать украсть куки
webView.evaluateJavaScript_completionHandler_(
    "document.cookie", 
    function(result, error) { 
        console.log("🍪 Cookies: " + result); 
    }
);
```

**Тест пройден**, если:

- Используется только `WKWebView`
- JavaScript включён только там, где нужен (и есть защита от XSS)
- Нет доступа к локальным файлам
- Все загрузки через HTTPS
- Нет моста к нативным методам без подтверждения

**Тест не пройден**, если:

- Найден `UIWebView`
- Можно через WebView прочитать файл `UserDefaults`
- Можно открыть `file://` и получить доступ к данным

-----

### MASTG-TEST-0077: Testing WebView Protocol Handlers

**О чём:** Проверяет кастомные протоколы WebView (например, `app://`). Нельзя ли через них получить доступ к файлам или вызвать опасные функции


Риски

**Path Traversal**  `app://../../../../etc/passwd` - Чтение любых файлов
**SSRF**    `app://localhost:8080/admin`  -- Доступ к внутренним сервисам
**Вызов методов**       `app://deleteUser?id=1` == Удаление данных
**Обход авторизации**       `app://adminPanel`   - Доступ к админке


например  - разработчик может создать свой протокол для WebView
вот так
```
// Пример: перехват всех запросов к app://
webView.configuration.setURLSchemeHandler(customHandler, forURLScheme: "app")

```

После этого в HTML внутри WebView можно написать:
```q
<img src="app://sensitive/avatar.jpg">
<a href="app://delete/user/1">Удалить</a>


----------------
Риск: Если обработчик написан плохо, можно:

- Прочитать любой файл на телефоне (app://../../private/var/...)
- Вызвать опасную функцию без подтверждения
- Обойти проверки безопасности
```

как тестировать:

```q
 найти кастомные протоколы
 
 
# Поиск регистрации схем
[0x00000000]> izz~setURLSchemeHandler
[0x00000000]> izz~URLSchemeHandler
[0x00000000]> izz~forURLScheme

# Поиск в Info.plist (если прописаны)
cat Info.plist | grep -A3 "CFBundleURLSchemes"

--------------


 найти реализацию обработчика

# Ищем класс, реализующий WKURLSchemeHandler
[0x00000000]> izz~WKURLSchemeHandler
[0x00000000]> izz~startURLSchemeTask
[0x00000000]> izz~stopURLSchemeTask

```

Подсосаться через Frida и попробовать загрузить разные URL:
```js
// hook_protocol_handler.js
var WKWebView = ObjC.classes.WKWebView;

// Отслеживаем загрузку в WebView
Interceptor.attach(WKWebView["- loadRequest:"].implementation, {
    onEnter: function(args) {
        var request = ObjC.Object(args[2]);
        var url = request.URL().absoluteString();
        console.log("[🌐] Loading: " + url);
        
        if (url.includes("://")) {
            var scheme = url.split("://")[0];
            if (["app", "local", "custom", "offline"].includes(scheme)) {
                console.log("[⚠️] Custom protocol detected: " + scheme);
            }
        }
    }
});




---------

или перехватить запрос к обработчику


// Более глубокий хук на кастомный протокол
var handlerClass = ObjC.classes.CustomURLSchemeHandler; // заменить на реальное имя
if (handlerClass) {
    var start = handlerClass["- startURLSchemeTask:"];
    Interceptor.attach(start.implementation, {
        onEnter: function(args) {
            var task = ObjC.Object(args[2]);
            var request = task.request();
            var url = request.URL().absoluteString();
            console.log("[🔴] Custom handler request: " + url);
            
            if (url.includes("..")) {
                console.log("[❌] PATH TRAVERSAL ATTEMPT!");
            }
        }
    });
}


```

либо вручную - если я знаю например - из info.plist схемы
прямо внутри webview искать... или вообще запустить бутфорс как-нибудь
```
app://../../../../etc/passwd
app://Library/UserDefaults.plist
app://Documents/secret.txt
local://../var/mobile/Containers/Data/Application/.../token.key
```

ну и пробуя разные варианты - смотреть - как приложение обрабатывает это все дело..


------
### MASTG-TEST-0078: Native Methods Exposed Through WebViews

**О чём:** Проверяет, не выставило ли приложение нативные методы через JavaScript-мост (`addJavascriptInterface` подобные). Через WebView можно вызвать эти методы

Проверить, можно ли через JavaScript внутри WebView вызвать нативный код приложения (Swift/ObjC) - например, чтобы украсть контакты, сделать платеж или удалить данные

то есть - если разработчил разрешил вебвью выполнять - какие -либо важные функции - 
например:
```swift
// Пример: разрешаем WebView вызывать функцию удаления чата
webView.configuration.userContentController.add(scriptMessageHandler, name: "deleteChat")
```

то есть вожможность для XSS
```html
Теперь в HTML внутри WebView можно написать:

<button onclick="window.webkit.messageHandlers.deleteChat.postMessage('123')">
    Удалить чат #123
</button>
```

КАК тестировать:
радар
```q

### найти зарегистрированные мосты

# Ищем добавление messageHandlers
[0x00000000]> izz~addScriptMessageHandler
[0x00000000]> izz~userContentController
[0x00000000]> izz~scriptMessageHandler

# Ищем конкретные имена мостов
[0x00000000]> izz~name:
# Ищем в том числе в Swift-метаданных
[0x00000000]> izz~MessageHandler

-----------

### найти реализацию обработчика

# Ищем класс, реализующий WKScriptMessageHandler
[0x00000000]> izz~didReceiveScriptMessage
[0x00000000]> izz~WKScriptMessageHandler
```


в динамике
Подсосаться через Frida и выполнить JavaScript для поиска мостов:
```js
// В консоли Frida после аттача к процессу
var webView = ObjC.classes.WKWebView.alloc().init();

// Попытка получить все доступные messageHandlers
webView.evaluateJavaScript_completionHandler_(`
    var handlers = [];
    if (window.webkit && window.webkit.messageHandlers) {
        for (var key in window.webkit.messageHandlers) {
            handlers.push(key);
        }
    }
    JSON.stringify(handlers);
`, function(result, error) {
    console.log("[🔍] Found messageHandlers: " + result);
});
```

**ест не пройден, если:**

- Можно через WebView вызвать `getContacts` → и получить массив контактов
- Можно вызвать `makePayment` → и деньги ушли без Face ID/пароля
- Можно вызвать `deleteAccount` → и аккаунт удалился

---------


### MASTG-TEST-0331: Use of Deprecated WebView APIs

**О чём:** Проверяет, не использует ли приложение старые, дырявые WebView (`UIWebView`). Нужно использовать современные `WKWebView`

WKWebView - гуд
UIWebView - не гуд

```q
# Поиск в бинарнике
[0x00000000]> izz~UIWebView
# Или через strings
strings MeetWay.app/MeetWay | grep -i "UIWebView"

----------

# Поиск импортов
[0x00000000]> ii~UIWebView
```

динамика
проверка через Frida (подтверждение)
```js
// find_webview.js
var UIApplication = ObjC.classes.UIApplication;
var keyWindow = UIApplication.sharedApplication().keyWindow();

function findWebViews(view) {
    if (view.isKindOfClass_(ObjC.classes.UIWebView)) {
        console.log("[🔴] CRITICAL: UIWebView found!");
    } else if (view.isKindOfClass_(ObjC.classes.WKWebView)) {
        console.log("[✅] WKWebView found (good)");
    }
    
    view.subviews().forEach(function(sub) {
        findWebViews(sub);
    });
}

findWebViews(keyWindow);
```


Если приложение всё ещё на `UIWebView` (редкость, но бывает в старых версиях):

1. UIWebView позволяет **полный доступ к файловой системе** при определённых настройках
2. Можно украсть `UserDefaults.plist` с токенами
3. Можно выполнить JavaScript в том же процессе (WKWebView - изолированный процесс, сложнее взломать)



----------

### MASTG-TEST-0332: Attacker-Controlled URI in WebViews

**О чём:** Проверяет, можно ли через внешнюю ссылку загрузить произвольный URL в WebView приложения (метод `loadRequest` с контролируемым параметром)

**Суть:** Проверить, можно ли заставить WebView загрузить **любой URL**, который контролирует злоумышленник (например, фишинговую страницу или страницу с вредоносным JavaScript)

Приложение открывает WebView и грузит туда URL, который приходит извне (из ссылки, из push-уведомления, из QR-кода, из данных, введённых пользователем)

**Риск:** Злоумышленник подсовывает свой URL: вместо `https://bank.com/pay` → `https://fake-bank.com/steal`

=========


 SAST
 найти опасные вызовы `loadRequest`
```
# Ищем загрузку URL в WebView
[0x00000000]> izz~loadRequest
[0x00000000]> izz~loadHTMLString
[0x00000000]> izz~loadFileURL

# Ищем источники внешних данных
[0x00000000]> izz~userActivity
[0x00000000]> izz~continueUserActivity
[0x00000000]> izz~openURL


 найти валидацию URL

# Ищем проверки белого списка
[0x00000000]> izz~host
[0x00000000]> izz~absoluteString
[0x00000000]> izz~hasPrefix


Хорошо, если есть:  
url.host == "myapp.com"
url.scheme == "https"


```

DAST

Если есть проверка домена, но плохая:
```
# Обходы проверки
https://myapp.com.evil.com
https://evil.com#myapp.com
https://myapp.com@evil.com
https://evil.com/myapp.com
https://myapp.com.evil.com/path
```

 ПРИМЕРЫ УЯЗВИМОСТИ:


```
плохо так:

func openWebView(with urlString: String) {
    let url = URL(string: urlString)!
    webView.loadRequest(URLRequest(url: url))
}

хорошо так :

func openWebView(with urlString: String) {
    guard let url = URL(string: urlString),
          let host = url.host,
          ["myapp.com", "help.myapp.com"].contains(host),
          url.scheme == "https" else { return }
    
    webView.loadRequest(URLRequest(url: url))
}

```

![[Снимок экрана 2026-04-29 в 21.01.29.png]]


-----

### MASTG-TEST-0333: Overly Broad File Read Access in WebViews

**О чём:** Проверяет, не может ли WebView читать **все** файлы приложения из-за неправильных настроек (например, `allowFileAccessFromFileURLs = true`).
Доступ должен быть только к определённой папке (обычно `Bundle` или `Documents`)

Если WebView может читать любые файлы:

- Украсть `UserDefaults.plist` (токены, настройки)
- Прочитать базу данных SQLite (сообщения, контакты)
- Слить ключи шифрования
- Прочитать cookies других доменов

тестить так:

```q
# Ищем конфигурацию WKWebView
[0x00000000]> izz~WKWebViewConfiguration
[0x00000000]> izz~preferences

# Опасные настройки
[0x00000000]> izz~allowFileAccessFromFileURLs
[0x00000000]> izz~allowUniversalAccessFromFileURLs

-----------


```

пример
```swift
// ❌ ОПАСНО (доступ ко всем файлам)
let config = WKWebViewConfiguration()
config.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")
config.preferences.setValue(true, forKey: "allowUniversalAccessFromFileURLs")

// ✅ БЕЗОПАСНО (или эти параметры отсутствуют, или false)
let config = WKWebViewConfiguration()
// allowFileAccessFromFileURLs по умолчанию false для WKWebView (но проверь!)
```

DAST
если есть возможность загрузить произвольный URL в WebView (тест 0332) или внедрить свой JavaScript (XSS)
то можно пробовать - прочитать файлы через веб вью

```js
// Пробуем прочитать системные файлы через file://
fetch('file:///private/var/mobile/Containers/Data/Application/.../Library/Preferences/com.apple.Preferences.plist')
fetch('file:///var/mobile/Library/Preferences/com.apple.Preferences.plist')

// Файлы приложения
fetch('file://' + window.location.pathname + '../../../../Library/UserDefaults.plist')
fetch('file://' + window.location.pathname + '../../../Documents/data.sqlite')
fetch('file://' + window.location.pathname + '../../../Library/Cookies/Cookies.binarycookies')
```

 Frida - перехватить чтение файлов

```js
// hook_file_access.js
var WKWebView = ObjC.classes.WKWebView;
var NSData = ObjC.classes.NSData;

// Хук на загрузку файлов в WebView
Interceptor.attach(WKWebView["- loadFileURL:allowingReadAccessToURL:"].implementation, {
    onEnter: function(args) {
        var fileURL = ObjC.Object(args[2]);
        var accessURL = ObjC.Object(args[3]);
        console.log(`[📂] Loading file: ${fileURL.absoluteString()}`);
        console.log(`[📂] Access granted to: ${accessURL.absoluteString()}`);
        
        // Опасно, если доступ даётся к корню или к папке выше
        if (accessURL.absoluteString() === "file:///" || 
            accessURL.path().split('/').length < 3) {
            console.log("[🔴] CRITICAL: Too broad file access!");
        }
    }
});

// Отслеживаем чтение данных через URLSession
var NSURLSession = ObjC.classes.NSURLSession;
Interceptor.attach(NSURLSession["- dataTaskWithURL:"].implementation, {
    onEnter: function(args) {
        var url = ObjC.Object(args[2]);
        if (url.absoluteString().startsWith("file://")) {
            console.log(`[🔴] FILE READ: ${url.absoluteString()}`);
        }
    }
});
```

```swift
так плохо
// Если эти строки есть в коде — проверяй их значения
config.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")

--------------------------

вот так должно быть
// Безопасная конфигурация
let config = WKWebViewConfiguration()
// Не трогаем allowFileAccessFromFileURLs (оставляем false по умолчанию)

// Если нужно дать доступ к файлам — только к конкретной папке
let documentsURL = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
webView.loadFileURL(fileURL, allowingReadAccessTo: documentsURL)
// А НЕ allowingReadAccessTo: URL(fileURLWithPath: "/")

```

![[Снимок экрана 2026-04-29 в 21.05.06.png]]


------------------

### MASTG-TEST-0335: WebView File Origin Access Relaxed

**О чём:** Проверяет, что доступ к файлам в WebView ограничен их происхождением. Нельзя ли из одной веб-страницы залезть в файлы другой
Это нарушает **политику одного источника (Same-Origin Policy)**

```q
Same-Origin Policy (SOP) — правило безопасности: код с сайта https://myapp.com не может читать данные с https://evil.com

В контексте файлов: HTML-страница из папки /Documents/page.html не должна читать файл из /Library/userdefaults.plist

Риски: Если эта защита ослаблена — одна страница может украсть данные другой
```

```q
# Ищем ослабление origin restrictions
[0x00000000]> izz~allowFileAccessFromFileURLs
[0x00000000]> izz~allowUniversalAccessFromFileURLs
[0x00000000]> izz~WKWebViewConfiguration

---------
Опасные значения (если true):

allowFileAccessFromFileURLs - разрешает XMLHttpRequest между файлами
 
allowUniversalAccessFromFileURLs - разрешает доступ к любым файлам независимо от origin
```

в коде
```swift
// ❌ ОПАСНО — ослабление origin restrictions
let config = WKWebViewConfiguration()
config.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")
config.preferences.setValue(true, forKey: "allowUniversalAccessFromFileURLs")

// ✅ БЕЗОПАСНО — оставляем по умолчанию (false)
let config = WKWebViewConfiguration()
```

сложнее 
DAST - проверить меж-файловые запросы
> Нужно иметь возможность загрузить два разных файла в WebView

**Шаг 1 - создать `page1.html`:**

```html
<html>
<body>
  <h1>Page 1</h1>
  <script>
    // Пытаемся прочитать другой файл
    fetch('file:///Documents/page2.html')
      .then(r => r.text())
      .then(data => console.log('Content of page2:', data));
  </script>
</body>
</html>
```
**Шаг 2 - загрузить `page1.html` в WebView**

**Шаг 3 - смотреть в логах, прочитался ли `page2.html`**

Если да - защита ослаблена (тест не пройден)


------------

### MASTG-TEST-0336: Runtime Setting of Relaxed File Origin Policies

**О чём:** Проверяет, что приложение **динамически** не меняет настройки безопасности WebView на более слабые во время работы


допустим, настроен WebView безопасно (все флаги = false), а потом в процессе работы приложение делает их true

статика
методы, меняющие настройки динамически
```q
# Ищем методы, которые могут менять конфигурацию
[0x00000000]> izz~setValue
[0x00000000]> izz~setPreferences
[0x00000000]> izz~setAllowFileAccess
[0x00000000]> izz~configuration

# Ищем вызовы configuration.preferences
[0x00000000]> izz~preferences

-------

Опасный паттерн: webView.configuration.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs") после создания WebView

-------

### найти JavaScript, который может менять настройки
# Ищем eval или строки, которые могут выполнить JavaScript
[0x00000000]> izz~evaluateJavaScript
[0x00000000]> izz~stringByEvaluatingJavaScript

```

пример
```q
// Изначально безопасно
let config = WKWebViewConfiguration()
let webView = WKWebView(frame: .zero, configuration: config)
// Потом в каком-то методе (например, после получения данных с сервера)
func relaxSecurity() {
    webView.configuration.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")
    webView.configuration.preferences.setValue(true, forKey: "allowUniversalAccessFromFileURLs")
}
```

  Frida для отслеживания изменений
```js
// track_runtime_changes.js
var WKPreferences = ObjC.classes.WKPreferences;

// Хуюк на setValue:forKey: (если настройки меняются через KVO)
Interceptor.attach(WKPreferences["- setValue:forKey:"].implementation, {
    onEnter: function(args) {
        var value = ObjC.Object(args[2]);
        var key = ObjC.Object(args[3]);
        
        if (key.toString().includes("FileAccess") || 
            key.toString().includes("UniversalAccess")) {
            console.log(`[⚠️] Runtime change: ${key} = ${value}`);
            console.log(`    Stack trace:`);
            var bt = Thread.backtrace(this.context, Backtracer.ACCURATE);
            bt.forEach(function(addr) {
                var mod = Process.findModuleByAddress(addr);
                if (mod && mod.name.includes("MeetWay")) {
                    console.log(`        ${mod.name} + 0x${addr.sub(mod.base).toString(16)}`);
                }
            });
        }
    }
});

// Хук хуюк - на evaluateJavaScript (если настройки меняются через JS)
var WKWebView = ObjC.classes.WKWebView;
Interceptor.attach(WKWebView["- evaluateJavaScript:completionHandler:"].implementation, {
    onEnter: function(args) {
        var js = ObjC.Object(args[2]);
        if (js.toString().includes("allowFileAccessFromFileURLs") ||
            js.toString().includes("allowUniversalAccess")) {
            console.log(`[🔴] Dangerous JS detected: ${js}`);
        }
    }
});
```

--------

##### вот - что необходимо внедрить (если этого нет) в мое приложение MeetWay
##### для того - чтобы я мог по настроящему протестировать все эти тесты  MASVS-PLATFORM

(подсказывает дипсик... не стал исправлять ниже - будет как дополнение к тому, что выше)

## 0056 — Sensitive Data Exposed via IPC

**Суть:** Проверить утечку данных через межпроцессное взаимодействие.

**Внедрить в MeetWay:**

- Кидать токен авторизации в `UIPasteboard.general`
    
- Создать `UserDefaults(suiteName:)` для обмена данными с расширениями
    
- Зарегистрировать кастомную URL-схему `meetway://`
   

---

## 0057 — Sensitive Data Disclosed Through User Interface

**Суть:** Проверить, не показывает ли приложение секреты на экране.

**Внедрить в MeetWay:**

- На экране чата отображать явный текст пароля или токен авторизации (не звёздочки)
    

---

## 0059 — Auto-Generated Screenshots for Sensitive Information

**Суть:** Проверить скриншот в App Switcher.

**Внедрить в MeetWay:**

- Не добавлять затемнение экрана в `applicationWillResignActive` — iOS сама сделает скриншот по умолчанию
    

---

## 0069 — Testing App Permissions

**Суть:** Проверить запрос разрешений.

**Внедрить в MeetWay:**

- Запросить доступ к контактам (без реальной необходимости)
    
- Запросить доступ к геолокации (без реальной необходимости)
    
- Добавить описания в `Info.plist`, но для лишних разрешений
    

---

## 0070 — Testing Universal Links

**Суть:** Проверить обработку универсальных ссылок.

**Внедрить в MeetWay:**

- Настроить файл `apple-app-site-association` на домене
    
- В методе `continueUserActivity` не проверять URL — загружать что пришло
    

---

## 0071 — Testing UIActivity Sharing

**Суть:** Проверить, что попадает в системное меню «Поделиться».

**Внедрить в MeetWay:**

- Добавить кнопку «Поделиться»
    
- В `activityItems` передавать токен авторизации
    

---

## 0072 — Testing App Extensions

**Суть:** Проверить расширения приложения.

**Внедрить в MeetWay:**

- Добавить Share Extension
    
- Расширение должно читать токен из `sharedDefaults`
    

---

## 0075 — Testing Custom URL Schemes

**Суть:** Проверить кастомные URL-схемы.

**Внедрить в MeetWay:**

- Схема `meetway://deleteChat?id=123` — удаляет чат без подтверждения
    
- Схема `meetway://sendMessage?text=xxx` — отправляет сообщение без подтверждения
    

---

## 0073, 0276–0280 — UIPasteboard (группа тестов на буфер обмена)

**0073 — Testing UIPasteboard (общий тест)**

- Использовать `UIPasteboard.general.string` для копирования данных
    

**0276 — Use of iOS General Pasteboard**

- Вызвать `UIPasteboard.general.string = token` (не приватную пасту)
    

**0277 — Sensitive Data in General Pasteboard at Runtime**

- При логине копировать пароль в общий буфер
    

**0278 — Pasteboard Contents Not Cleared After Use**

- После вставки пароля не очищать буфер (`general.string = ""` не вызывать)
    

**0279 — Pasteboard Contents Not Expiring**

- Не ставить таймаут на очистку — данные висят вечно
    

**0280 — Pasteboard Contents Not Restricted to Local Device**

- Оставить iCloud-синхронизацию включённой (по умолчанию)
    

---

## 0290 — Runtime Verification of Sensitive Content Exposure in Screenshots During App Backgrounding

**Суть:** Динамическая проверка скриншотов при уходе в фон.

**Внедрить в MeetWay:**

- Совпадает с 0059 — затемнение экрана отсутствует
    

---

## 0076–0078, 0331–0336 — WebView (группа тестов)

**0076 — Testing iOS WebViews (общий)**

- Добавить экран с WKWebView, загружающий URL из внешнего параметра
    

**0077 — WebView Protocol Handlers**

- Зарегистрировать кастомный протокол `app://`
    
- Через него открывать доступ к файлам приложения
    

**0078 — Native Methods Exposed Through WebViews**

- Добавить JavaScript-мост `deleteChat` и `getContacts`
    
- Использовать `addScriptMessageHandler` без проверок
    

**0331 — Use of Deprecated WebView APIs**

- Не внедрять (Apple запретила UIWebView). Использовать WKWebView — это норма.
    

**0332 — Attacker-Controlled URI in WebViews**

- В WebView вызывать `loadRequest(URLRequest(url: externalUrl))`
    
- Без валидации URL
    

**0333 — Overly Broad File Read Access**

- В WKWebViewConfiguration включить:  
    `allowFileAccessFromFileURLs = true`
    

**0335 — WebView File Origin Access Relaxed**

- В WKWebViewConfiguration включить:  
    `allowUniversalAccessFromFileURLs = true`
    

**0336 — Runtime Setting of Relaxed File Origin Policies**

- Где-то в коде (по таймеру или по URL) динамически переключить настройки через `setValue:forKey:`

