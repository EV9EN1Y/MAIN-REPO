
==`dig` (Domain Information Groper)== -  это мощнейшая консольная утилита для опроса DNS-серверов. Если `nslookup` - это простой "молоток", то `dig` - это "швейцарский нож" с кучей лезвий. Он не просто показывает IP-адрес, а выдает полный, структурированный отчет о DNS-запросе: технические флаги, время ответа, TTL (время жизни записи в кеше) и путь запроса


### задачи


- **Разведка (Reconnaissance):** Находить поддомены, виртуальные хосты и IP-адреса, которые не видны на основном сайте. Это ключевой этап в определении поверхности атаки (ASVS)
   
- **Верификация:** Проверять, корректно ли настроены DNS-записи (MX, TXT, SPF) для защиты от фишинга и спуфинга
   
- **Диагностика:** Трассировать путь DNS-запроса, чтобы найти, где именно происходит сбой или кеширование

--------

### этичность

**С точки зрения использования - инструмент абсолютно безопасен.** Это просто легальный способ спросить у DNS-сервера информацию. Ты не ломаешь чужие системы, а лишь собираешь публичные данные. **Однако есть три важных нюанса:**

1. **Соблюдай границы (Scope).** Если в контракте четко указано тестировать только `www.example.com`, то использование `dig` для поиска поддоменов (`admin.example.com`) может считаться выходом за рамки разрешенной деятельности. Всегда согласовывай это с заказчиком
   
2. **Не оставляй следов.** Активный перебор имен (например, через `dig` в скрипте) генерирует много запросов и оставляет следы в DNS-логах и на серверах, что может быть расценено как DoS-атака или разведка. В профессиональной среде это "шумно", и тебя могут задетектить
   
3. **Это приманка для злоумышленников.** Взломщики используют `dig` для разведки внутри взломанных систем, особенно в Kubernetes (ищут внутренние сервисы типа `*.svc.cluster.local`)
4. Поэтому в защищенных средах его часто блокируют. Для тебя это значит, что его использование нужно документировать и согласовывать, чтобы не вызвать подозрений у службы безопасности компании, которая мониторит такие запросы


----------

#### `dig` (основной инструмент)

- **Обычный запрос (A-запись):** `dig example.com
    
- **Запрос конкретного типа записи (MX, TXT):** `dig example.com MX
    
- **Запрос к конкретному DNS-серверу:** `dig @8.8.8.8 example.com`
    
- **Получить только IP-адрес (без "воды"):** `dig +short example.com`
    
- **Обратный запрос (IP -> домен):** `dig -x 8.8.8.8
    
- **Трассировка пути запроса (от корневых серверов):** `dig +trace example.com Это покажет, как запрос доходит до авторитативного сервера, что бывает очень полезно при поиске проблем

---------

# установка на мак

1. **  
    Проверь:** Открой **Терминал** (Applications → Utilities → Terminal) и введи команду:
  
   ```c
   brew install bind
   
   ---------
   
   
   dig -v
   ```

#  Базовые команды 

**1. Обычный запрос (A-запись):**  
Показывает IP-адрес домена.


```c
dig example.com
```

Выдаст полный "разговор" с сервером, включая версию, флаги и статистику

------

**2. Краткий ответ (только IP):**  
Идеально для скриптов и когда нужен только чистый результат

```c
dig example.com +short
```

Вернет только IP-адрес: `93.184.216.34


-----

**3. Запрос конкретной записи:**

- **MX (Mail Exchanger):**

```c
dig example.com MX
```
    
- **TXT (текстовые записи, например, SPF/DKIM):**
 
  ```c
   dig example.com TXT
  ```

или все сразу
```q
for type in A AAAA MX TXT CNAME NS SOA CAA SRV NAPTR DS DNSKEY RRSIG NSEC LOC HINFO RP; do
    echo "### $type ###"
    dig example.com $type +short
    echo ""
done
```

```q



A	dig example.com A	IPv4-              адрес сервера

AAAA	dig example.com AAAA	IPv6-        адрес сервера

MX	dig example.com MX	                Почтовые серверы (Mail Exchangers)

TXT	dig example.com TXT	             Текстовые записи (SPF, DKIM, DMARC, верификация)

CNAME	dig example.com CNAME	          Алиас (псевдоним) на другой домен

NS	dig example.com NS	                Авторитативные DNS-серверы (Name Servers)

SOA	dig example.com SOA	             Техническая информация о зоне (Start of Authority)

PTR	dig -x IP-                        адрес	Обратная запись — домен по IP (Pointer)

CAA	dig example.com CAA	             Разрешенные центры сертификации (Certificate Authority Authorization)

SRV	dig example.com SRV	             Сервисные записи (для VoIP, Active Directory)

NAPTR	dig example.com NAPTR	          Маршрутизация служб (VoIP, SIP)

DS	dig example.com DS	                Записи для DNSSEC (Delegation Signer)

DNSKEY	dig example.com DNSKEY	       Публичные ключи DNSSEC

RRSIG	dig example.com RRSIG	          Цифровые подписи DNSSEC

NSEC	dig example.com NSEC	Next Secure  отрицательный ответ в DNSSEC

LOC	dig example.com LOC	             Географическое местоположение

HINFO	dig example.com HINFO	          Информация о хосте (CPU, OS) — сейчас почти не используется

RP	dig example.com RP	                Контактное лицо, ответственное за домен

AFSDB	dig example.com AFSDB	          Для AFS-файловых систем

X25	dig example.com X25	             Для сетей X.25 (редкость)

ISDN	dig example.com ISDN	             Для ISDN-адресов (редкость)

RT	dig example.com RT	Route Through — маршрутизация

NSAP	dig example.com NSAP	NSAP-        адреса (устаревшее)

SIG	dig example.com SIG	             Подпись (устаревший аналог RRSIG)

```




----------

**4. Запрос к конкретному DNS-серверу:**  
Позволяет обойти системный кеш и опросить, например, Google Public DNS (`8.8.8.8`)

```c
dig @8.8.8.8 example.com
```


---------


**5. Трассировка пути запроса (+trace):**  
Показывает полный путь DNS-запроса от корневых серверов до авторитативного[](https://scansearch.net/en/articles/nslookup-dig-dns-query-tools/).
```c
dig example.com +trace
```

Это незаменимо для поиска проблем с делегированием DNS.

----------

**6. Обратный запрос (поиск домена по IP):**

```c
dig -x 8.8.8.8
```

--------



                          ‼️

---------------

# П Р И М Е Р - реал данные скрыты




##  DNS Reconnaissance - Пример анализа поверхности атаки

Данный репозиторий содержит учебный пример проведения DNS-разведки в рамках тестирования безопасности веб-приложений (ASVS)

**Цель:** Демонстрация методов обнаружения скрытых поддоменов, виртуальных хостов и IP-адресов

**Инструменты:** `dig` (macOS), Google Public DNS, авторитативный DNS-сервер

**Важно:** Все данные в этом репозитории являются **анонимизированными**. Реальные IP-адреса и доменные имена были изменены для защиты конфиденциальной информации. Данный материал предназначен исключительно для образовательных целей

**Выводы:**

- Обнаружена Round-Robin балансировка между тремя серверами
   
- Отсутствие MX-записей снижает поверхность атаки
   
- Домен хостится на Wix (SaaS-платформа), что накладывает ограничения на пентест
   
- Стандартные поддомены не зарегистрированы, требуется словарный перебор

-------

демонстрация методов анализа поверхности атаки
Инструмент: dig 




----------


# ---------- 1. Базовый A-запрос ----------
```c
➡️   $ dig example.com


; <<>> DiG 9.10.6 <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 61852
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:  (данные изменены на несуществующие)
example.com.		3600	IN	A	185.230.63.172   
example.com.		3600	IN	A	185.230.63.185   
example.com.		3600	IN	A	185.230.63.108   

;; Query time: 96 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Fri Aug 07 11:54:01 +05 2026
;; MSG SIZE  rcvd: 102
```
###### Интерпретация: Три A-записи указывают на Round-Robin DNS
###### Это балансировка нагрузки между тремя серверами. Для пентеста
###### важно потом просканировать все три IP, так как их конфигурация может
###### отличаться (версии ПО, открытые порты, патчи)





----------



# ---------- 2. Запрос через Google Public DNS ----------

```c
➡️   $ dig @8.8.8.8 example.com
(данные изменены на несуществующие)

; <<>> DiG 9.10.6 <<>> @8.8.8.8 example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56428
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		3600	IN	A	185.230.63.108   # ← изменено
example.com.		3600	IN	A	185.230.63.185   # ← изменено
example.com.		3600	IN	A	185.230.63.172   # ← изменено

;; Query time: 151 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Aug 07 11:54:10 +05 2026
;; MSG SIZE  rcvd: 102
```
###### Интерпретация: Запрос через внешний DNS подтверждает,
###### что данные консистентны. Разный порядок IP-адресов подтверждает
###### работу механизма Round-Robin. Никаких признаков DNS-спуфинга






----------




# ---------- 3. Попытка трассировки (не сработала) ----
```c
➡️   $ dig example.com +trace

(данные изменены на несуществующие)

; <<>> DiG 9.10.6 <<>> example.com +trace
;; global options: +cmd
;; Received 28 bytes from 192.168.0.1#53(192.168.0.1) in 2 ms
```
###### Интерпретация: Локальный DNS-резолвер работает в режиме
###### forwarder и не возвращает полную цепочку запросов. Для точной
###### трассировки нужно обращаться напрямую к авторитативному DNS




---------




# ---------- 4. Запрос MX-записей ----------

```c
➡️   $ dig example.com MX
(данные изменены на несуществующие)

; <<>> DiG 9.10.6 <<>> example.com MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16734
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	MX

;; AUTHORITY SECTION:
example.com.		1672	IN	SOA	ns12.wixdns.net. support.wix.com. 2017042009 10800 3600 1209600 3600

;; Query time: 47 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Fri Aug 07 11:54:21 +05 2026
;; MSG SIZE  rcvd: 117
```
###### Интерпретация: MX-записи отсутствуют. Это означает, что
###### домен не настроен для приема почты. Поверхность атаки меньше
###### SOA-запись указывает на Wix-хостинг (ns12.wixdns.net)




---------



# ---------- 5. Запрос напрямую к авторитативному DNS -


```c
➡️   $ dig @ns12.wixdns.net example.com
(данные изменены на несуществующие)

; <<>> DiG 9.10.6 <<>> @ns12.wixdns.net example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50097
;; flags: qr aa rd; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;example.com.			IN	A

;; ANSWER SECTION:
example.com.		3600	IN	A	185.230.63.185   
example.com.		3600	IN	A	185.230.63.108   
example.com.		3600	IN	A	185.230.63.172  

;; Query time: 46 msec
;; SERVER: 216.239.36.101#53(216.239.36.101)
;; WHEN: Fri Aug 07 11:55:27 +05 2026
;; MSG SIZE  rcvd: 102
```
###### Интерпретация: Запрос к авторитативному серверу - самый
###### достоверный источник. Флаг "aa" (Authoritative Answer) подтверждает,
###### что это оригинальные данные, а не кеш. Подтверждены три IP-адреса



-----------


# ---------- 6. Запрос ANY-записей ----------


```c
➡️   $ dig example.com ANY
(данные изменены на несуществующие)
; <<>> DiG 9.10.6 <<>> example.com ANY
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21289
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	ANY

;; ANSWER SECTION:
example.com.		3497	IN	A	185.230.63.185   
example.com.		3497	IN	A	185.230.63.108   
example.com.		3497	IN	A	185.230.63.172 
  
;; Query time: 3 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Fri Aug 07 11:55:44 +05 2026
;; MSG SIZE  rcvd: 102
```
###### Интерпретация: Запрос ANY вернул только A-записи
###### Другие типы записей (MX, CNAME, NS) отсутствуют
###### Домен минималистичный - только веб-хостинг



--------------


# ---------- 7. Запрос TXT-записей ----------

```c
➡️   $ dig example.com TXT

(данные изменены на несуществующие)

; <<>> DiG 9.10.6 <<>> example.com TXT
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64781
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;example.com.			IN	TXT

;; ANSWER SECTION:
example.com.		3600	IN	TXT	"google-site-verification=XXXXXX_FAKE_XXXXXX"

;; Query time: 99 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Fri Aug 07 11:55:47 +05 2026
;; MSG SIZE  rcvd: 135
```
###### Интерпретация: TXT-запись содержит метку подтверждения
###### владения доменом для Google Search Console. Это штатная
###### техническая запись, угроз безопасности не представляет



---------------

# ---------- 8. Перебор поддоменов (стандартные имена) 

```c
(данные изменены на несуществующие)
➡️   $ dig admin.example.com
➡️   $ dig mail.example.com
➡️   $ dig dev.example.com
➡️   $ dig staging.example.com
```
# (Сокращенный вывод для всех четырех)
```c
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: XXXX
;; QUESTION SECTION:
;admin.example.com.		IN	

;; AUTHORITY SECTION:
example.com.		1578	IN	SOA	ns12.wixdns.net. support.wix.com. 2017042009 10800 3600 1209600 3600
```
###### Интерпретация: Все проверенные поддомены (admin, mail,
###### dev, staging) не существуют (NXDOMAIN). Это говорит о том, что
###### либо дополнительные сервисы не используются, либо они скрыты
###### под нестандартными именами (например, portal, backend, management).
###### Для полного покрытия требуется словарный перебор




-----------



# ---------- 9. Обратный DNS-запрос (PTR) ----------


```c
➡️   $ dig -x 185.230.63.172   # ← изменен последний октет
(данные изменены на несуществующие)
; <<>> DiG 9.10.6 <<>> -x 185.230.63.172

;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13024
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232

;; QUESTION SECTION:
;172.63.230.185.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
172.63.230.185.in-addr.arpa. 1800 IN	PTR	unalocated.63.wixsite.com.


;; Query time: 368 msec
;; SERVER: 192.168.0.1#53(192.168.0.1)
;; WHEN: Fri Aug 07 11:56:00 +05 2026
;; MSG SIZE  rcvd: 95
```
###### Интерпретация: PTR-запись указывает на технический домен
###### Wix (wixsite.com), а не на example.com. Это нормально для
###### shared hosting - владелец домена не контролирует обратные записи
###### Этот метод не позволяет найти другие домены на том же IP
