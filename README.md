🔥🕸️ рекомендую скачать обсидиан и открывать данные файлы через него
        таким образом каждая статья будет в цвете и структурированна лучше, чем на гитхабе


🟣 главная папка -> (ПУТЬ-В-КИБЕР-БЕЗ)


-> ВНУТРИ: (полный список смотри в конце READMI ⇩⇩⇩)

🔶 решение лаб по WEB с отчетами / пейлоадами и скриптами + теория по тематикам лаб

🔶 база по сетям (OSI, TCP/IP, DNS, HTTP...)

🔶 криптография

🔶 база по ОС

🔶 база JavaScript

🔶 база Python

🔶 база HTML/CSS // SQL/XML


ВСЕ СТАТЬИ И ИССЛЕДОВАНИЯ ПРЕДСТАВЛЕННЫ ИСКЛЮЧИТЕЛЬНО В ИНОФОРМАЦИОННЫХ ЦЕЛЯХ ДЛЯ ОБУЧЕНИЯ ЗАЩИТЕ ОТ КИБЕР АТАК, ВСЕ ЧТО ЗДЕСЬ СОДЕРЖИТСЯ - ПРИМЕНЯЛОСЬ ЛИБО НА МОИХ СОБСТВЕННЫХ ПРОЕКТАХ, ЛИБО В СПЕЦ ЛАБОЛАТОРИЯХ. ИСПОЛЬЗОВАТЬ ДАННУЮ ИНФОРМАЦИЮ НА РЕАЛЬНЫХ РЕСУРСАХ ЗАПРЕЩЕНО ЗАКОНОМ

| Ст. 272 УК РФ | Ст. 272.1 УК РФ | Ст. 273 УК РФ | (GDPR, COPPA, ФЗ-152) |


АВТОР КОНТЕНТА НЕ НЕСЕТ НИКАКОЙ ОТВЕТСТВЕННОСТИ И НЕ ОТВЕЧАЕТ ЗА ЛЮБОЕ НЕПРАВИЛЬНОЕ ИСПОЛЬЗОВАНИЕ ИЛИ УЩЕРБ, ПРИЧИНЕННЫЙ ИСПОЛЬЗОВАНИЕМ ДАННЫХ ОТЧЕТОВ И СКРИПТОВ

----------------------------------------
## **СТРУКТУРА РЕПОЗИТОРИЯ**

### 🔥 **ЛАБЫ PortSwigger + ТЕОРИЯ**



**🔥 КВИНТЭССЕНЦИЯ_всех_лаб** / ВЫЖИМКА_КАЖДОЙ РЕШЕННОЙ_ЛАБЫ_САМА_СУТЬ.md - шпаргалка-конспект по всем типам атак с видами, примерами и защитами
(по лабам портсвиггер)



**Access Control (14 лаб)**

- теория + iDOR
    
- вертикальное, горизонтальное повышение прав
    
- GUID, IDOR
   

**Authentication & OAuth 2.0 (20+ лаб)**

- уязвимости на основе пароля (5)
    
- 2FA уязвимости (3)
    
- атаки на сброс пароля
    
- HTTP host header атаки
    
- OAuth 2.0 (4 лабы: CSRF, redirect_uri, Open Redirect)
   

**Business Logic (12 лаб)**

- теория
    
- уязвимости на email
    
- 12 практических лаб
   

**GraphQL (5 лаб + шпаргалка)**

- теория
    
- скрытые запросы
    
- обход brute force защиты
    
- CSRF через GraphQL
   

**Insecure Deserialization (8 лаб)**

- PHP object injection
    
- PHP 7 уязвимости
    
- RCE через Java (Apache, ysoserial)
    
- Ruby, Java
   

**JWT (9 лаб)**

- теория
    
- Hashcat брутфорс
    
- Jwk injection
    
- alg: none атака
    
- Jku атака
    
- Kid + Path Traversal
    
- Algorithm Confusion (2 лабы)
   

**OS Injection (5 лаб)**

- теория
    
- OS command injection (5 лаб)
   

**Path Traversal (6 лаб)**

- теория
    
- Path traversal (6 лаб)
   

**Race Conditions (6 лаб)**

- теория
    
- Race Conditions (6 лаб)
   

**SQL Injection (16 лаб)**

- выводы
    
- 15 практических лаб + теория
   

**SSRF (7 лаб)**

- теория
    
- SSRF с редиректом
    
- Blind SSRF + Shellshock
    
- SSRF парсеры
   

**Web Ai LLM (4 лабы)**

- теория
    
- LLM уязвимости (4 лабы)
   

**Web Cache Deception (5 лаб)**

- теория
    
- WCD (5 лаб)
   

**XML/XXE (9 лаб)**

- теория
    
- XXE (9 лаб)
   

**XSS (база + лаборатории)**

- CSP
    
- DOM XSS
    
- Reflected XSS
    
- Stored XSS
    
- теория
   

---

### 🏛️ **ФУНДАМЕНТАЛЬНЫЕ ЗНАНИЯ**

**Сети**

- OSI, OSI 7, TCP, UDP, IP, NAT
    
- DNS, HTTP, WAF
    
- SOCKET, PORTs
    
- Tor, VPN
    
- Web Cache
    
- Анатомия веб-запроса
   

**Криптография**

- PKI-сертификаты
    
- X.509
    
- Hash функции
    
- SSL/TLS рукопожатия
    
- E2EE (сквозное шифрование)
    
- Асимметричное и симметричное шифрование
   

**Базы данных**

- SQL база
    
- XML
    
- Базы данных (общее)
   

**Языки программирования**

- JavaScript (база + Secure + payloads)
    
- Python (база + типы данных)
    
- HTML/CSS
    
- SQL

**API Testing (5 лаб)**

- теория
    
- брутфорс путей, OpenAPI/Swagger
    
- Server-Side Parameter Pollution (SSPP)
    
- Mass Assignment, переполнение integer
    
- Path Traversal к legacy endpoints


**CORS (3 лабы)**

- теория
    
- Origin reflection
    
- Null origin whitelist
    
- XSS + trusted subdomain

**CSRF (12 лаб)**

- теория
    
- No defenses / token missing
    
- Token validation depends on method / presence
    
- Token not tied to user session / tied to non-session cookie
    
- Double submit + CRLF injection
    
- SameSite Lax bypass (method override, cookie refresh)
    
- SameSite Strict bypass (client-side redirect, sibling domain)
    
- Referer validation bypass (missing header, substring check)



   **Clickjacking (5 лаб)**

- теория
    
- Basic overlay (iframe + button)
    
- Prefilled form via GET
    
- Frame buster bypass (sandbox)
    
- DOM-based XSS + clickjacking
    
- Multistep clickjacking
    

**HTTP Host Header Vulnerabilities (5 лаб)**

- теория
    
- Authentication bypass (localhost)
    
- Web cache poisoning (double Host)
    
- Routing-based SSRF
    
- Flawed request parsing
    
- Connection state attack


**HTTP Request Smuggling (12 лаб)**

- теория
    
- CL.TE / [TE.CL](https://te.cl/)
    
- TE.TE obfuscated
    
- Differential responses (CL.TE / [TE.CL](https://te.cl/))
    
- Bypass front-end controls (CL.TE / [TE.CL](https://te.cl/))
    
- Reveal front-end rewriting
    
- Capture other users' requests
    
- Deliver reflected XSS
    
- H2.TE queue poisoning
    
- [H2.CL](https://h2.cl/) JS smuggling
    

**Information Disclosure (5 лаб)**

- теория
    
- Error messages (stack traces)
    
- Debug pages (phpinfo)
    
- Backup files (.bak, .git)
    
- Custom headers (X-Custom-IP-Authorization)
    
- Version control history

**SSTI (Server-Side Template Injection) (5 лаб)**

- теория
    
- Ruby (ERB) RCE
    
- Python (Tornado, Django) RCE / info disclosure
    
- Java (Freemarker) RCE
    
- Node (Handlebars) RCE


**Prototype Pollution (10 лаб)**

- теория
    
- Client-side DOM XSS (URL, constructor, flawed sanitization)
    
- Third-party library gadget
    
- Server-side privilege escalation
    
- Server-side RCE (NODE_OPTIONS, shell+input)
    
- Detection without reflection
    
- Constructor prototype bypass


**Web Cache Poisoning (11 лаб)**

- теория
    
- Unkeyed header (X-Forwarded-Host)
    
- Unkeyed cookie
    
- Multiple headers
    
- Targeted User-Agent
    
- Unkeyed query string / parameter
    
- Parameter cloaking
    
- Fat GET
    
- Normalization mismatch (404)
    
- DOM XSS via external resource
    
- Combining vulnerabilities
    

**WebSocket (3 лабы)**

- теория
    
- Message manipulation (XSS)
    
- Handshake manipulation (X-Forwarded-For)
    
- CSWSH (Cross-Site WebSocket Hijacking)
    

**Операционные системы**

- Linux архитектура
    
- Windows архитектура
    
- OS command
    
- POINT RECON


   

**Браузеры**

- Browser DOM, BOM, JS теория
   

---

### 🛡️ **AppSec**

- MASTG
    
- MASVS
   

---

### 📝 **Инструменты**

- общий список ссылок 
   

---

### 🔍 **Аудиты**

- Аудит безопасности соц-сети MeetWay (чуть позже начну)
    
- план проверки
   

---

### 💼 **Собеседования**

- разбор вопросов для собеса
    
- вопросы на Web & App Pentest / AppSec
   

---


ВСЕ СТАТЬИ И ИССЛЕДОВАНИЯ ПРЕДСТАВЛЕННЫ ИСКЛЮЧИТЕЛЬНО В ИНОФОРМАЦИОННЫХ ЦЕЛЯХ ДЛЯ ОБУЧЕНИЯ ЗАЩИТЕ ОТ КИБЕР АТАК, ВСЕ ЧТО ЗДЕСЬ СОДЕРЖИТСЯ - ПРИМЕНЯЛОСЬ ЛИБО НА МОИХ СОБСТВЕННЫХ ПРОЕКТАХ, ЛИБО В СПЕЦ ЛАБОЛАТОРИЯХ. ИСПОЛЬЗОВАТЬ ДАННУЮ ИНФОРМАЦИЮ НА РЕАЛЬНЫХ РЕСУРСАХ ЗАПРЕЩЕНО ЗАКОНОМ

| Ст. 272 УК РФ | Ст. 272.1 УК РФ | Ст. 273 УК РФ | (GDPR, COPPA, ФЗ-152) |

АВТОР КОНТЕНТА НЕ НЕСЕТ НИКАКОЙ ОТВЕТСТВЕННОСТИ И НЕ ОТВЕЧАЕТ ЗА ЛЮБОЕ НЕПРАВИЛЬНОЕ ИСПОЛЬЗОВАНИЕ ИЛИ УЩЕРБ, ПРИЧИНЕННЫЙ ИСПОЛЬЗОВАНИЕМ ДАННЫХ ОТЧЕТОВ И СКРИПТОВ
