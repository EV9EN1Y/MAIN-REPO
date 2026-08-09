#### testssl.sh это бесплатная консольная утилита для проверки поддержки TLS/SSL шифров, протоколов и криптографических уязвимостей на серверах

------
**Базовый запуск**  
```c
testssl.sh example.com
```
Проверяет хост на порту 443

### запуск всего набора комнад
```q
testssl.sh -U example.com

------

скорость и задержки
testssl.sh -U --connect-timeout 15 --openssl-timeout 10 example.com

--connect-timeout <сек>
 Устанавливает таймаут для TCP-подключения. Это позволяет не ждать слишком долго, если сервер не отвечает, и переходить к следующей цели
 
--openssl-timeout <сек> 
Ограничивает время выполнения команд OpenSSL s_client. Это полезно для предотвращения зависаний на этапе установки защищенного соединения

--socket-timeout <сек>
   Контролирует таймаут для операций с сокетами (подключение, отправка/получение данных), что помогает бороться с "залипающими" сокетами

```

---------

**Проверка других портов**  
```c
testssl.sh example.com:8443

testssl.sh example.com:465

testssl.sh --starttls smtp example.com:25
```

**Основные параметры**  

```q
-t, --starttls <protocol> – проверка STARTTLS (smtp, pop3, imap, ftp) 
 
-p, --protocols – только поддерживаемые TLS протоколы  

-s, --std, --standard – стандартные наборы шифров  

-e, --each-cipher – все доступные шифры (~370)  

-E, --cipher-per-proto – шифры по каждому протоколу  

-U, --vulnerable, --vulnerabilities – все известные уязвимости  

-H, --heartbleed – Heartbleed  

-R, --renegotiation – Renegotiation  

-B, --breach – BREACH  

-O, --poodle – POODLE  

-W, --sweet32 – SWEET32  

-4, --rc4 – RC4 шифры  

-h, --header – HTTP заголовки безопасности  

-c, --client-simulation – симуляция клиентов  

--ip <IP> – принудительный IP  

--wide – детальный вывод  

--mapping <openssl|iana> – схема именования шифров  

--show-each – все проверенные шифры  

--add-ca <cafile> – свои корневые сертификаты

```

**Таймауты и масс-скан**  
```q
--connect-timeout <seconds> – таймаут TCP  

--openssl-timeout <seconds> – таймаут openssl  

--file <fname> или -iL <fname> – цели из файла  

--mode <serial|parallel> – режим масс-сканирования
```

**Вывод**  
```q
--jsonfile <file> – JSON  

--logfile <file> – текстовый лог
```

**Примеры**  
```q
testssl.sh https://example.com 
testssl.sh example.com:8443
testssl.sh --starttls smtp mail.example.com:25
testssl.sh -p -U example.com
testssl.sh --file targets.txt --mode parallel  
testssl.sh --jsonfile report.json example.com
```

----------

