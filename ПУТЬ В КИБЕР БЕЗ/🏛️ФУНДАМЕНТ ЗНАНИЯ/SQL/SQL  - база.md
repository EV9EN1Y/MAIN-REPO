
изучаю по курсу от https://sql-academy.org/ru/guide





Structured Query Language

есть внешние ключ - это уник id параметр в таблице
есть внутр ключ - это id от других таблиц

==КАК ПОНЯТЬ КАКОЙ ДИАЛЕКТ У БД? 
![[Снимок-14.57.06.png]]
Эксплуатация бд строится по такому принципу:

1.Обнаружение** (поиск уязвимой точки)
2. Определение типа СУБД** (с помощью комбинации тестов из таблицы выше).
3. Разведка структуры** (через **информационные схемы**):
    
    MySQL/PostgreSQL**: Используют `information_schema.tables` и `information_schema.columns
        
    MS SQL Server: `information_schema.tables` или `sysobjects`.
        
    Oracle: `all_tables`, `all_tab_columns`.




==ВЫВОД  -------  в PostgreSQL 17.5 
```sql
           //PostgreSQL 17.5 
//Вывод произвольных значений -  SELECT 
SELECT 'Hello world'F    - это как принт

//Вывод Для вывода всех полей из таблицы *
 SELECT * FROM FamilyMembers
 
//Вывод данных из определённых колонок
 SELECT idUser, nameUser FROM nameTAble

//Вывод  псевдонимы - такой же вывод но колонка со своим именем
SELECT idUser, nameUser AS newName FROM nameTAble

Вывод или так без as
SELECT idUser, nameUser newName FROM nameTAble
```

==ЛИТЕРАЛЫ --------          PostgreSQL 17.5 
Основными типами литералов в SQL являются
- строковый  ---->   'это строка'
- числовой
- логический
- NULL
- литерал даты и времени
- (") используются для идентификаторов (имен таблиц, столбцов)
- "\" (экранирующий символ)
- "\n" перенос строки
- E - 

```sql
// (вывод с преобразованием)
SELECT 'Строка Другая строка' as String 
// (вывод с преобразованием + перенос строки)
SELECT 'Строка \n Другая строка' as String
// арифметика (Result = 2)
SELECT (5 * 2 - 6) / 2 AS Result;

```
|Оператор|Описание|Пример|
|:-:|:--|:--|
|%|Деление по модулю|11 % 5 = 1|
|*|Умножение|10 * 16 = 160|
|+|Сложение|98 + 2 = 100|
|-|Вычитание|50 - 51 = -1|
|/|Деление|1 / 2 = 0.5|

==Литералы даты и времени
```sql
// показать все FamilyMembers где birthday старше '1970-12-30'
SELECT * FROM FamilyMembers WHERE birthday > '1970-12-30'
```
![[Снимок-14.45.36.png]]


==TRUE и FALSE, означающие истинность и ошибочность какого-либо утверждения.

==Значение NULL означает "нет данных", "нет значения". значит значения нет вообще , даже нет пустой строки и нет даже пробела, нил - это пустота

```sql
// вывыд в верхнем регистре UPPER
SELECT UPPER('Hello world') AS upper_string;

//Извлекает часть даты (год, месяц, день и т.д.) для указанной даты
SELECT EXTRACT(YEAR FROM DATE '2022-06-16') AS year;

// использование функ =LENGTH=, в полях name из tableName выводим countCharName = число символов в строке  
SELECT name, LENGTH(name) AS countCharName FROM tableName;

// совмещение функций  UPPER и LEFT
LEFT - вывела все буквы, UPPER берез 3 первые бук и ставит в верх регистр
SELECT UPPER(LEFT('текст-такой', 3)) AS str; (str = тек)

```


# Справочник по функциям MySQL

*Строковые функции
[CHAR](https://sql-academy.org/ru/handbook/mysql/char)[CONCAT](https://sql-academy.org/ru/handbook/mysql/concat)[INSTR](https://sql-academy.org/ru/handbook/mysql/instr)[LEFT](https://sql-academy.org/ru/handbook/mysql/left)[LENGTH](https://sql-academy.org/ru/handbook/mysql/length)[LOCATE](https://sql-academy.org/ru/handbook/mysql/locate)[LOWER](https://sql-academy.org/ru/handbook/mysql/lower)[LPAD](https://sql-academy.org/ru/handbook/mysql/lpad)[LTRIM](https://sql-academy.org/ru/handbook/mysql/ltrim)[REPEAT](https://sql-academy.org/ru/handbook/mysql/repeat)[REPLACE](https://sql-academy.org/ru/handbook/mysql/replace)[REVERSE](https://sql-academy.org/ru/handbook/mysql/reverse)[RIGHT](https://sql-academy.org/ru/handbook/mysql/right)[RPAD](https://sql-academy.org/ru/handbook/mysql/rpad)[RTRIM](https://sql-academy.org/ru/handbook/mysql/rtrim)[SUBSTRING](https://sql-academy.org/ru/handbook/mysql/substring)[TRIM](https://sql-academy.org/ru/handbook/mysql/trim)[UPPER](https://sql-academy.org/ru/handbook/mysql/upper)
*Числовые функции
[ABS](https://sql-academy.org/ru/handbook/mysql/abs)[CEILING](https://sql-academy.org/ru/handbook/mysql/ceiling)[COS](https://sql-academy.org/ru/handbook/mysql/cos)[EXP](https://sql-academy.org/ru/handbook/mysql/exp)[FLOOR](https://sql-academy.org/ru/handbook/mysql/floor)[GREATEST](https://sql-academy.org/ru/handbook/mysql/greatest)[LEAST](https://sql-academy.org/ru/handbook/mysql/least)[LOG](https://sql-academy.org/ru/handbook/mysql/log)[MOD](https://sql-academy.org/ru/handbook/mysql/mod)[PI](https://sql-academy.org/ru/handbook/mysql/pi)[POW](https://sql-academy.org/ru/handbook/mysql/pow)[RAND](https://sql-academy.org/ru/handbook/mysql/rand)[ROUND](https://sql-academy.org/ru/handbook/mysql/round)[SIGN](https://sql-academy.org/ru/handbook/mysql/sign)[SIN](https://sql-academy.org/ru/handbook/mysql/sin)[SQRT](https://sql-academy.org/ru/handbook/mysql/sqrt)[TAN](https://sql-academy.org/ru/handbook/mysql/tan)[TRUNCATE](https://sql-academy.org/ru/handbook/mysql/truncate)

*Функции дат и времени
[ADDDATE](https://sql-academy.org/ru/handbook/mysql/adddate)[CURDATE](https://sql-academy.org/ru/handbook/mysql/curdate)[CURTIME](https://sql-academy.org/ru/handbook/mysql/curtime)[DATE](https://sql-academy.org/ru/handbook/mysql/date)[DATE_ADD](https://sql-academy.org/ru/handbook/mysql/date_add)[DATE_FORMAT](https://sql-academy.org/ru/handbook/mysql/date_format)[DATE_SUB](https://sql-academy.org/ru/handbook/mysql/date_sub)[DATEDIFF](https://sql-academy.org/ru/handbook/mysql/datediff)[DAY](https://sql-academy.org/ru/handbook/mysql/day)[DAYNAME](https://sql-academy.org/ru/handbook/mysql/dayname)[DAYOFMONTH](https://sql-academy.org/ru/handbook/mysql/dayofmonth)[DAYOFWEEK](https://sql-academy.org/ru/handbook/mysql/dayofweek)[DAYOFYEAR](https://sql-academy.org/ru/handbook/mysql/dayofyear)[FROM_UNIXTIME](https://sql-academy.org/ru/handbook/mysql/from_unixtime)[HOUR](https://sql-academy.org/ru/handbook/mysql/hour)[MAKEDATE](https://sql-academy.org/ru/handbook/mysql/makedate)[MAKETIME](https://sql-academy.org/ru/handbook/mysql/maketime)[MICROSECOND](https://sql-academy.org/ru/handbook/mysql/microsecond)[MINUTE](https://sql-academy.org/ru/handbook/mysql/minute)[MONTH](https://sql-academy.org/ru/handbook/mysql/month)[MONTHNAME](https://sql-academy.org/ru/handbook/mysql/monthname)[NOW](https://sql-academy.org/ru/handbook/mysql/now)[QUARTER](https://sql-academy.org/ru/handbook/mysql/quarter)[SEC_TO_TIME](https://sql-academy.org/ru/handbook/mysql/sec_to_time)[SECOND](https://sql-academy.org/ru/handbook/mysql/second)[STR_TO_DATE](https://sql-academy.org/ru/handbook/mysql/str_to_date)[SUBDATE](https://sql-academy.org/ru/handbook/mysql/subdate)[TIME_TO_SEC](https://sql-academy.org/ru/handbook/mysql/time_to_sec)[TIMEDIFF](https://sql-academy.org/ru/handbook/mysql/timediff)[TIMESTAMPADD](https://sql-academy.org/ru/handbook/mysql/timestampadd)[TIMESTAMPDIFF](https://sql-academy.org/ru/handbook/mysql/timestampdiff)[UNIX_TIMESTAMP](https://sql-academy.org/ru/handbook/mysql/unix_timestamp)[WEEK](https://sql-academy.org/ru/handbook/mysql/week)[YEAR](https://sql-academy.org/ru/handbook/mysql/year)
*Оконные функции
[DENSE_RANK](https://sql-academy.org/ru/handbook/mysql/dense_rank)[FIRST_VALUE](https://sql-academy.org/ru/handbook/mysql/first_value)[LAG](https://sql-academy.org/ru/handbook/mysql/lag)[LAST_VALUE](https://sql-academy.org/ru/handbook/mysql/last_value)[LEAD](https://sql-academy.org/ru/handbook/mysql/lead)[RANK](https://sql-academy.org/ru/handbook/mysql/rank)[ROW_NUMBER](https://sql-academy.org/ru/handbook/mysql/row_number)
*Агрегатные функции
[AVG](https://sql-academy.org/ru/handbook/mysql/avg)[COUNT](https://sql-academy.org/ru/handbook/mysql/count)[GROUP_CONCAT](https://sql-academy.org/ru/handbook/mysql/group_concat)[MAX](https://sql-academy.org/ru/handbook/mysql/max)[MIN](https://sql-academy.org/ru/handbook/mysql/min)[SUM](https://sql-academy.org/ru/handbook/mysql/sum)
*Функции регулярных выражений
[REGEXP_INSTR](https://sql-academy.org/ru/handbook/mysql/regexp_instr)[REGEXP_LIKE](https://sql-academy.org/ru/handbook/mysql/regexp_like)[REGEXP_REPLACE](https://sql-academy.org/ru/handbook/mysql/regexp_replace)[REGEXP_SUBSTR](https://sql-academy.org/ru/handbook/mysql/regexp_substr)
*Продвинутые функции
[CAST](https://sql-academy.org/ru/handbook/mysql/cast)[COALESCE](https://sql-academy.org/ru/handbook/mysql/coalesce)[IF](https://sql-academy.org/ru/handbook/mysql/if)[IFNULL](https://sql-academy.org/ru/handbook/mysql/ifnull)[ISNULL](https://sql-academy.org/ru/handbook/mysql/isnull)[NULLIF](https://sql-academy.org/ru/handbook/mysql/nullif)[WITH](https://sql-academy.org/ru/handbook/mysql/with)



# Справочник по функциям PostgreSQL
*# Строковые функции
[CHR](https://sql-academy.org/ru/handbook/postgresql/chr)[CONCAT](https://sql-academy.org/ru/handbook/postgresql/concat)[LENGTH](https://sql-academy.org/ru/handbook/postgresql/length)[LOWER](https://sql-academy.org/ru/handbook/postgresql/lower)[LPAD](https://sql-academy.org/ru/handbook/postgresql/lpad)[LTRIM](https://sql-academy.org/ru/handbook/postgresql/ltrim)[POSITION](https://sql-academy.org/ru/handbook/postgresql/position)[REPEAT](https://sql-academy.org/ru/handbook/postgresql/repeat)[REPLACE](https://sql-academy.org/ru/handbook/postgresql/replace)[REVERSE](https://sql-academy.org/ru/handbook/postgresql/reverse)[RPAD](https://sql-academy.org/ru/handbook/postgresql/rpad)[RTRIM](https://sql-academy.org/ru/handbook/postgresql/rtrim)[SUBSTRING](https://sql-academy.org/ru/handbook/postgresql/substring)[TRIM](https://sql-academy.org/ru/handbook/postgresql/trim)[UPPER](https://sql-academy.org/ru/handbook/postgresql/upper)
*# Числовые функции
[ABS](https://sql-academy.org/ru/handbook/postgresql/abs)[CEIL](https://sql-academy.org/ru/handbook/postgresql/ceil)[COS](https://sql-academy.org/ru/handbook/postgresql/cos)[EXP](https://sql-academy.org/ru/handbook/postgresql/exp)[FLOOR](https://sql-academy.org/ru/handbook/postgresql/floor)[GREATEST](https://sql-academy.org/ru/handbook/postgresql/greatest)[LEAST](https://sql-academy.org/ru/handbook/postgresql/least)[LOG](https://sql-academy.org/ru/handbook/postgresql/log)[MOD](https://sql-academy.org/ru/handbook/postgresql/mod)[PI](https://sql-academy.org/ru/handbook/postgresql/pi)[POWER](https://sql-academy.org/ru/handbook/postgresql/power)[RANDOM](https://sql-academy.org/ru/handbook/postgresql/random)[ROUND](https://sql-academy.org/ru/handbook/postgresql/round)[SIGN](https://sql-academy.org/ru/handbook/postgresql/sign)[SIN](https://sql-academy.org/ru/handbook/postgresql/sin)[SQRT](https://sql-academy.org/ru/handbook/postgresql/sqrt)[TAN](https://sql-academy.org/ru/handbook/postgresql/tan)[TRUNC](https://sql-academy.org/ru/handbook/postgresql/trunc)
###### Функции дат и времени
[AGE](https://sql-academy.org/ru/handbook/postgresql/age)[CURRENT_DATE](https://sql-academy.org/ru/handbook/postgresql/current_date)[CURRENT_TIME](https://sql-academy.org/ru/handbook/postgresql/current_time)[DATE](https://sql-academy.org/ru/handbook/postgresql/date)[DATE_PART](https://sql-academy.org/ru/handbook/postgresql/date_part)[DATE_TRUNC](https://sql-academy.org/ru/handbook/postgresql/date_trunc)[EXTRACT](https://sql-academy.org/ru/handbook/postgresql/extract)[MAKE_DATE](https://sql-academy.org/ru/handbook/postgresql/make_date)[MAKE_INTERVAL](https://sql-academy.org/ru/handbook/postgresql/make_interval)[MAKE_TIME](https://sql-academy.org/ru/handbook/postgresql/make_time)[NOW](https://sql-academy.org/ru/handbook/postgresql/now)[TO_CHAR](https://sql-academy.org/ru/handbook/postgresql/to_char)[TO_DATE](https://sql-academy.org/ru/handbook/postgresql/to_date)[TO_TIMESTAMP](https://sql-academy.org/ru/handbook/postgresql/to_timestamp)
###### Оконные функции
[DENSE_RANK](https://sql-academy.org/ru/handbook/postgresql/dense_rank)[FIRST_VALUE](https://sql-academy.org/ru/handbook/postgresql/first_value)[LAG](https://sql-academy.org/ru/handbook/postgresql/lag)[LAST_VALUE](https://sql-academy.org/ru/handbook/postgresql/last_value)[LEAD](https://sql-academy.org/ru/handbook/postgresql/lead)[RANK](https://sql-academy.org/ru/handbook/postgresql/rank)[ROW_NUMBER](https://sql-academy.org/ru/handbook/postgresql/row_number)
###### Агрегатные функции
[AVG](https://sql-academy.org/ru/handbook/postgresql/avg)[COUNT](https://sql-academy.org/ru/handbook/postgresql/count)[MAX](https://sql-academy.org/ru/handbook/postgresql/max)[MIN](https://sql-academy.org/ru/handbook/postgresql/min)[STRING_AGG](https://sql-academy.org/ru/handbook/postgresql/string_agg)[SUM](https://sql-academy.org/ru/handbook/postgresql/sum)
###### Продвинутые функции
[CAST](https://sql-academy.org/ru/handbook/postgresql/cast)[COALESCE](https://sql-academy.org/ru/handbook/postgresql/coalesce)[NULLIF](https://sql-academy.org/ru/handbook/postgresql/nullif)[WITH](https://sql-academy.org/ru/handbook/postgresql/with)


```sql
SELECT member_name, 
    LENGTH(member_name) - POSITION(' ', member_name)  AS lastname_length FROM FamilyMembers;
    
member_name - столбцы имен фамилий
LENGTH(member_name) - число сиволов в каждом member_name)
POSITION(' ', member_name) - позиция пробела по счету в каждомmember_name
lastname_length - число симв в фамилии
```


## операtор DISTINCT - убирает повторяющиеся данные 

```sql
// получаем только уникальные данные
SELECT DISTINCT class FROM Student_in_class;
```

## операtор  WHERE где - для выборки
## операtор  WHERE AND где - для выборки + AND
```sql
SELECT * FROM Student
WHERE first_name = 'Grigorij' AND EXTRACT(YEAR FROM birthday) > 2000;
```

![[Снимок-16.18.54.png]]
### Результатом сравнения любого значения с NULL является NULL


## Логические операторы
1. ==AND==  и — оба условия должны быть верны
2. ==OR==  или— достаточно, чтобы выполнилось хотя бы одно условие или оба сразу
3. ==NOT==  не— условие становится противоположным (SELECT * FROM Trip WHERE NOT town_to = 'Moscow'; все рейсы которые не прилетают в москву
)
4. ==XOR== только в MySQL (только что=то одно выполнилось) — это оператор, который помогает выбрать строки, где выполняется только одно из двух условий, но не оба сразу

<mark style="background:#fff88f">> [!В PostgreSQL нет оператора XOR, поэтому используется комбинация AND и OR для достижения того же результата.] </mark>

## порядок логичеких опереаторов 
- Сначала — ==NOT
- Затем — ==AND
- Потом — ==XOR
- В конце — ==OR

```sql
//вывести все данные из таблицы но где есть поля has_kitchen и  has_internet
select *
from Rooms
where has_kitchen = true and has_internet = true
```

==операторы==
### IS NULL  
позволяет узнать, равно ли проверяемое значение NULL, т.е. пустое ли значение.
```sql
select first_name, last_name 
from Student
where middle_name is null
```
### BETWEEN min AND max 
позволяет узнать, расположено ли проверяемое значение столбца в интервале между min и max, включая сами значения min и max. 
тоже самое что и : ```sql WHERE field >= min AND field <= max ```
```sql
SELECT * FROM Payments
WHERE unit_price BETWEEN 100 AND 500;
```

## IN 
позволяет узнать, входит ли проверяемое значение столбца в список определённых значений
```sql
SELECT * FROM FamilyMembers
WHERE status IN ('father', 'mother');

// все записи из таб где год из др = 2000 или 2002 или 2004
SELECT *
FROM Student
WHERE YEAR(birthday) IN (2000, 2002, 2004);
```

## LIKE 
(==поиск по одному шиблону==)используется при условных запросах, когда мы хотим узнать, соответствует ли строка определённому шаблону , используем с % или с __
![[Снимок-17.13.22.png]]
```sql
// ищем те почты где есть @hotmail.  
// знаки % это обезка любых символов % 
SELECT name, email FROM Users
WHERE email LIKE '%@hotmail.%'

// примеры , регистр не важен
... WHERE поле_таблицы LIKE 'text%' // начин с text
... WHERE поле_таблицы LIKE '%text' // заканчив с text
... WHERE поле_таблицы LIKE '_ext' // начин с _(любой символ) но после ext
... WHERE поле_таблицы LIKE 'begin%end' // нач begin конец end

// пример 
SELECT member_name FROM FamilyMembers
where member_name LIKE '%иванов%'
```

## ESCAPE   -символ используется с LIKE
для экранирования специальных символов (% и _ ) _В случае если вам нужно найти строки, содержащие эти специальные символы как обычный текст, вы можете использовать ESCAPE-символ.

```sql
  // без ESCAPE - найдет `Любые символы` + `"50"` + `Любые символы`.
	SELECT * FROM products WHERE description LIKE '%50%%';
	
	// с ESCAPE - Найдёт только строки, содержащие `"50%"`
	SELECT * FROM products WHERE description LIKE '%50\%%' ESCAPE '\';
```


## REGEXP (или его синоним     RLIKE )
(==поиск по нескольким шиблонам==)используется для поиска и обработки строковых данных с помощью регулярных выражений, для сложных шаблонов поиска, которые трудно реализовать с помощью оператора LIKE.
==поиск по нескольким условиям или использование специальных символов и диапазонов а LIKE  там где одно условие = один шиблон==
регисронезависимый!

==/== - для экранирования таких символов ==., *, +, ?, [, ], (, ), {, }, |, \ )== 
![[Снимок-18.23.39.png]]
```sql
// где  начало (^) с John
SELECT * FROM Users WHERE name REGEXP '^John'  (JohnTravolta)

// название которых оканчивается ($) на букву «e» или «y»: (History)
SELECT * FROM  Subject WHERE name REGEXP '[ey]$' 

// где заканчивается ($) на  outlook.com или (|)  icloud.com
SELECT * FROM Users WHERE email REGEXP '@(outlook.com|icloud.com)$'

// не содержит 2 и 8 
//[^28] обозначает любой символ, кроме «2» и «8»
SELECT * FROM Users WHERE phone_number REGEXP '^[^28]*$'

// номера начинающиеся с +7
// \\ - нужно для экранирования +, так как + это спец символ
SELECT name, phone_number FROM Users WHERE phone_number REGEXP '^\\+7'

// все юзеры чей майл заканчивается на @outlook.com или @live.com
select name, email from Users
where email REGEXP '@(outlook.com|live.com)$'
```
## ORDER BY  --------------- сортировка
==ASC (по возрастанию) и DESC(по убыван) - направление сортировки:==
(если это бул тип то при ASC сперва идет false потом true, а NULL значения идут последними)
```sql
SELECT name FROM Company ORDER BY name;// name в алфавитном порядке
```
==сортировка по нескольким столбцам==
```sql
...ORDER BY столбец_1 [ASC | DESC], столбец_2 [ASC | DESC];

//  strField по порядку, numField по поряд, dateField в обратном порядке
SELECT * FROM Table ORDER BY 
  strField ASC, numField ASC, dateField DESC
```

```sql 
// вывести все табл, где в поле  member_name есть Quincey
// и отсортировать сперва status по убыв и потом также member_name
select * from FamilyMembers
where member_name like '%Quincey%'
order by 
status asc, member_name asc
```

вне курса
# CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END
CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE 'a' END

это конструкция (конекртно oracle) 
кейс  кода (1=1) тогда (1/0) иначе "x" конец

то есть рабоатет так, проверяем условие WHEN и если оно true 
тогда выполним блок THEN 
а если WHEN = false - то выполнится esle

и спользовать можно для вызова ошибок
помещаем внутрь WHEN (условие) и потом блок THEN где (1 делить на 0)
и если блок WHEN будет true успех то выполнится блок который вызовет ошибку и запрос упадет!



# CAST ()
```sql
CAST('123' AS int)   → 123 (число)
CAST('text' AS int)  → ОШИБКА (нельзя преобразовать) эту ошибку можно прочитать если она выводится! так еще и эта ошибка может выглядеть так: // "123@password2025" невозможно перобразовать в int - то есть покажет сам пароль!
CAST((SELECT 'password') AS int)``` 
*так как пароль - это строковый тип почти всегда - тогда можно попытать преобразовать его в int - что вызовет ошибку и если ошибка выводится то мы увидим результат!*
*САМА ОШИБКА МОЖЕТ ВЕРНУТЬ ТО ЗНАЧЕНИЕ КОТОРОЕ МЫ ПЫТАЕМСЯ ПЕРЕВЕСТИ В ИНТ
и если сделать CAST((SELECT 'password') AS int) то ошибка высветит сам пароль и скажет что его нельзя в инт перевести!