
## база по работе браузера  тут ->  [[theory]]

#### Источники

Источник - это свойство JavaScript, которое принимает данные, которые потенциально контролируются злоумышленником. Примером источника является свойство`location.search`, потому что оно считывает входные данные из строки запроса, что относительно просто для злоумышленника. В конечном счете, любое свойство, которое может контролироваться злоумышленником, является потенциальным источником. Сюда входят ссылающийся URL-адрес (отображенный строкой `document.referrer`), файлы cookie пользователя (отображенные строкой `document.cookie`) и веб-сообщения.

#### Приемники

Приемник - это потенциально опасная функция JavaScript или DOM-объект, который может вызвать нежелательные эффекты при передаче данных, контролируемых злоумышленником. Например, функция eval() является приемником, поскольку она обрабатывает аргумент, который передается ей как JavaScript. Примером приемника HTML является document.body.innerHTML, поскольку он потенциально позволяет злоумышленнику внедрять вредоносный HTML-код и выполнять произвольный JavaScript

наиболее распространенным источником является URL-адрес, к которому обычно осуществляется с помощью объекта `location`

Наиболее распространенным источником для DOM XSS является URL-адрес, к которому обычно осуществляется с помощью объекта `window.location`.

###### Как проверить межсайтовый скриптинг на основе DOM
нужно использовать браузер с инструментами разработчика, такими как Chrome. Вам нужно проработать каждый доступный источник по очереди и протестировать каждый из них индивидуально




основные источники где можно тырить данные
```
document.URL 
document.documentURI 
document.URLUnencoded 
document.baseURI 
location 
document.cookie 
document.referrer
 window.name 
 history.pushState 
 history.replaceState 
 localStorage 
 sessionStorage 
 IndexedDB (mozIndexedDB, webkitIndexedDB, msIndexedDB) 
 Database
 
 
 некоторые из основных приемников, которые могут привести к уязвимостям DOM-XSS
 
 document.write() 
 document.writeln() 
 document.domain 
 element.innerHTML 
 element.outerHTML 
 element.insertAdjacentHTML 
 element.onevent
 
 
 функции jQuery также являются приемниками, которые могут привести к уязвимостям DOM-XSS
 
 add() 
 after() 
 append() 
 animate() 
 insertAfter() 
 insertBefore() 
 before() 
 html() 
 prepend() 
 replaceAll() 
 replaceWith() 
 wrap() 
 wrapInner() 
 wrapAll() 
 has() 
 constructor() 
 init() 
 index() 
 jQuery.parseHTML() 
 $.parseHTML()
 
 
```

причем, на основе DOM есть много видов уязвимостей!
таблица с портсвиггер

| Уязвимость на основе DOM                                                                                                   | Пример раковины            |
| -------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| ЛАБОРАТОРИИ [DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based)                                 | `document.write()`         |
| [Открытые](https://portswigger.net/web-security/dom-based/open-redirection) ЛАБОРАТОРИИ перенаправления                    | `window.location`          |
| ЛАБОРАТОРИИ [манипуляций с печеньем](https://portswigger.net/web-security/dom-based/cookie-manipulation)                   | `document.cookie`          |
| [Инъекция JavaScript](https://portswigger.net/web-security/dom-based/javascript-injection)                                 | `eval()`                   |
| [Манипуляция доменом документов](https://portswigger.net/web-security/dom-based/document-domain-manipulation)              | `document.domain`          |
| [Отравление WebSocket-URL](https://portswigger.net/web-security/dom-based/websocket-url-poisoning)                         | `WebSocket()`              |
| [Манипуляция ссылками](https://portswigger.net/web-security/dom-based/link-manipulation)                                   | `element.src`              |
| [Манипуляция веб-сообщениями](https://portswigger.net/web-security/dom-based/web-message-manipulation)                     | `postMessage()`            |
| [Манипулирование заголовком запроса Ajax](https://portswigger.net/web-security/dom-based/ajax-request-header-manipulation) | `setRequestHeader()`       |
| [Локальное манипулирование путь к файлу](https://portswigger.net/web-security/dom-based/local-file-path-manipulation)      | `FileReader.readAsText()`  |
| [SQL-инъекция на стороне клиента](https://portswigger.net/web-security/dom-based/client-side-sql-injection)                | `ExecuteSql()`             |
| [Манипуляция с хранением HTML5](https://portswigger.net/web-security/dom-based/html5-storage-manipulation)                 | `sessionStorage.setItem()` |
| [Внедрение XPath на стороне клиента](https://portswigger.net/web-security/dom-based/client-side-xpath-injection)           | `document.evaluate()`      |
| [Инъекция JSON на стороне клиента](https://portswigger.net/web-security/dom-based/client-side-json-injection)              | `JSON.parse()`             |
| [Манипулирование данными DOM](https://portswigger.net/web-security/dom-based/dom-data-manipulation)                        | `element.setAttribute()`   |
| [Отказ в обслуживании](https://portswigger.net/web-security/dom-based/denial-of-service)                                   | `RegExp()`                 |

----

#### сахарок      DOM Invader

здесь решил лабу DOM XSS  обычно + с помощью инвайдера 
->> [[01_DOM_XSS+document_write+DOM-invader]]

автоматически ищет **DOM-based XSS** и другие клиентские уязвимости

смысл такой, можно вручную вбить проверочный пейлоад - и потом искать свой пейлоад внутри js и html кода перебирая тонны строк кода

это все можно автоматизировать через DOM Invader - штука, встроенная прямо в браузер Burp

 DOM Invader == работает как дополнительная вкладка в инструментах разработчика (F12). Ты включаешь его, вводишь специальное слово-маркер (canary), например "pentest", и просто бродишь по сайту. DOM Invader смотрит, попало ли твое слово в какие-то опасные места (типа innerHTML, document.write, eval), и если да — **подсвечивает это** и показывает стек вызовов, где именно в коде это произошло


бесплатный и мощный:

1. Открываешь встроенный браузер Burp'а (Proxy → Open Browser)
2. Включаешь DOM Invader (иконка расширения)
3. Открываешь F12 → вкладка DOM Invader
4. Ищешь уязвимости руками, но с подсветкой от инструмента

---

==в браузере хромиум есть встроеное расширение DOM Invader

настройки

### Главные настройки (Settings)

Тут базовые тумблеры:

- **Enable DOM Invader** — включить/выключить зверя (должен быть ON)
    
- **Canary** — твое уникальное слово-маркер (например, "pentest123"). DOM Invader будет вставлять это слово везде и смотреть, где оно всплывет
    
- **Logging** — настройки логирования (включай всё, чтобы ничего не пропустить)
    

## Два режима работы

### Режим 1: Ты сам пихаешь слово (ручной)

Ты копируешь canary (кнопка **Copy canary**) и сам вставляешь его:

- В URL параметры (`?q=hackme123`)
    
- В поля форм
    
- В любые инпуты на странице 
    

DOM Invader просто **смотрит**, куда это слово попало и что с ним случилось

### Режим 2: DOM Invader сам пихает слово (автоматический) — ТО, ЧТО ТЫ ЗАМЕТИЛ!

В настройках есть опция **"Inject canary into all sources"**Когда она включена, DOM Invader сам:

- **Добавляет твоё слово во все URL параметры** (кнопка "Inject URL params") 
    
- **Заполняет им все формы** (кнопка "Inject forms") 
    
- **Пихает во все возможные источники данных** на странице 
    

Причём для каждого источника он добавляет **уникальный суффикс**, чтобы ты потом мог понять, через какой именно параметр слово попало в уязвимое место

есть чекбокс "Inject canary into all sources" и рядом шестеренка для тонкой настройки — можно указать, какие именно параметры трогать, а какие игнорить

### Типы атак (Attack Types) — **САМОЕ ВАЖНОЕ!**

Тут ты выбираешь, какие уязвимости искать:

- **DOM XSS** — ищем классический DOM-based XSS (вставляет canary в разные источники и смотрит, не попал ли он в опасные приемники типа innerHTML, eval и т.д.)
    
- **Prototype Pollution** — ищем прототип поллюшн (это когда можно засрать глобальный Object.prototype и через это выполнить код). DOM Invader тестирует и источники (где можно вставить говно), и гаджеты (как это говно превратить в атаку)
    
- **PostMessage** — ищем уязвимости в postMessage (когда вкладки общаются между собой). DOM Invader может логировать все сообщения, перехватывать их и автоматически тестить с canary
    
- **DOM Clobbering** — ищем уязвимости, когда HTML-элементы переопределяют глобальные JS-переменные (например, `<div id=location>` может перекрыть настоящий `window.location`)
    

### Misc (Разное)

Тут всякая вспомогательная куйня:

- **Web Messages** — список всех перехваченных postMessage-сообщений
    
- **Sinks** — список всех найденных опасных приемников (куда можно вставить говно)
    
- **Settings для конкретных sink'ов** — например, можно настроить, какие атрибуты тестировать (onload, onclick, src и т.д.)
    

---

## Как этим пользоваться (короткая инструкция)

1. **Включаешь DOM Invader** (главная вкладка)
2. **Ставишь canary** (например, "xss-123")
3. **Идешь на целевой сайт** и просто бродишь по нему, тыкаешь кнопки, заполняешь формы
4. **Заглядываешь во вкладки DOM Invader** — если canary где-то засветился в опасном месте, тебе подсветят
5. **Если нашел** — смотришь вкладку Sinks, там будет путь (stack trace) прямо в код, где это говно происходит





