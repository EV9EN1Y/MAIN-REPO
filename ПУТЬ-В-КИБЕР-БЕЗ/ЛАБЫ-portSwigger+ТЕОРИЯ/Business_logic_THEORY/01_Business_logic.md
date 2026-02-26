## Чрезмерное доверие к управлению со стороны клиента

лаба - https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-excessive-trust-in-client-side-controls

есть уязвимость позволяющая покупать товары по цене несоотвт оригинальной
нужно купить "Легкую кожаную куртку l33t"

------

в каталоге товаров 
при выборе товара и добавлении в корзину отпр запрос 

```http
POST /cart HTTP/2
Host: 0a89005204a871e6855a3ab700170001.web-security-academy.net
Cookie: session=yuJEGljHYqOhummLbqlLqgAdDyFdweox
Content-Length: 49
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0a89005204a871e6855a3ab700170001.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a89005204a871e6855a3ab700170001.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

productId=1&redir=PRODUCT&quantity=1&price=133700
```


в запросе указана цена

пробую ее поменять
добавил кучу курток по цене 1 бакс

![[bissines01010101.png]]

ну и я смог это оплатить 

лаба решена

выводы очевидны

простейший перехват - поменял цену