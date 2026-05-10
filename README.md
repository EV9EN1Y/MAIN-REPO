# iOS Dev → AppSec → Mobile & Web Application Security Research

## 👨‍💻 Обо мне

**iOS разработчик - 2 года опыта.**

Выпустил 3 собственных приложения в App Store, включая:
- iOS-социальную сеть **MeetWay**
- полностью свой **Back-end** для неё

После разработки соцсети:
- провёл её полный аудит по мобильной безопасности
- настроил защиту на основе всех пройденных тестов OWASP MASVS
- взломал свою же защиту (проверка на прочность)
- разобрался в том, как сделать такую защиту -которую будет ну очень сложно сломать

Настроил Ci/CD пайплайн

На сервере Cloud.ru  развёрнул *self-hosted раннер GitHub Actions*. Раннер зарегистрировал с тегом `self-hosted`, он залочен на конкретный репозиторий и настроен как системный сервис с автозапуском.  

В репозиторий добавлен файл пайплайна `.github/workflows/test.yml`, который при пуше в ветку `main` запускает задание на этом раннере. 

Задание выполняется в изолированном Docker-контейнере (`alpine:latest`) в не-привилегированном режиме, выводит тестовое сообщение и возвращает статус **Success**.  

CI/CD полностью работоспособен: GitHub управляет пайплайном, раннер на [Cloud.ru] его исполняет, контейнер создаётся под каждую задачу и уничтожается после выполнения

**Результат:** замкнутый полный цикл «атакую -> защищаю -> аудирую» на реальном проекте [[0_MeetWay]]

---

## ⚠️ Дисклеймер

ВСЕ СТАТЬИ И ИССЛЕДОВАНИЯ ПРЕДСТАВЛЕНЫ ИСКЛЮЧИТЕЛЬНО В ИНФОРМАЦИОННЫХ ЦЕЛЯХ ДЛЯ ОБУЧЕНИЯ ЗАЩИТЕ ОТ КИБЕРАТАК.

ВСЁ, ЧТО ЗДЕСЬ СОДЕРЖИТСЯ, ПРИМЕНЯЛОСЬ ЛИБО НА МОИХ СОБСТВЕННЫХ ПРОЕКТАХ, ЛИБО В СПЕЦИАЛЬНЫХ ЛАБОРАТОРИЯХ.

ИСПОЛЬЗОВАТЬ ДАННУЮ ИНФОРМАЦИЮ НА РЕАЛЬНЫХ РЕСУРСАХ ЗАПРЕЩЕНО ЗАКОНОМ.

Ст. 272 УК РФ | Ст. 272.1 УК РФ | Ст. 273 УК РФ | GDPR | COPPA | ФЗ-152

АВТОР КОНТЕНТА НЕ НЕСЕТ НИКАКОЙ ОТВЕТСТВЕННОСТИ ЗА ЛЮБОЕ НЕПРАВИЛЬНОЕ ИСПОЛЬЗОВАНИЕ ИЛИ УЩЕРБ.

---

## 📊 Общая статистика репозитория

- Лабораторные PortSwigger (Web) - **238+**
- Тесты по мобильной безопасности iOS (MASVS) - **65+**
- Темы / артефакты - **полное покрытие от фундамента до CI-CD**

---

## 🕸️ WEB Security (PortSwigger + теория)

**Решено лабораторных: 238+**

### Access Control - 14 лаб
теория / iDOR / вертикальное / горизонтальное / GUID / IDOR

### Authentication & OAuth 2.0 - 20+ лаб
пароль / 2FA / сброс пароля / HTTP host header / OAuth 2.0 (CSRF, redirect_uri, Open Redirect)

### Business Logic - 12 лаб
теория / email / 12 практик

### GraphQL - 5 лаб + шпаргалка
теория / скрытые запросы / обход brute force / CSRF

### Insecure Deserialization - 8 лаб
PHP object injection / PHP7 / RCE Java (Apache, ysoserial) / Ruby / Java

### JWT - 9 лаб
теория / Hashcat брутфорс / Jwk injection / alg:none / Jku / Kid+Path Traversal / Algorithm Confusion (2)

### OS Injection - 5 лаб
теория / OS command injection (5)

### Path Traversal - 6 лаб
теория / Path traversal (6)

### Race Conditions - 6 лаб
теория / Race Conditions (6)

### SQL Injection - 16 лаб
выводы / 15 практик + теория

### SSRF - 7 лаб
теория / редирект / Blind + Shellshock / парсеры

### Web Ai LLM - 4 лабы
теория / LLM (4)

### Web Cache Deception - 5 лаб
теория / WCD (5)

### XML / XXE - 9 лаб
теория / XXE (9)

### XSS (база + лаборатории) - 35 лаб
CSP / DOM XSS / Reflected XSS / Stored XSS / теория

 

### API Testing (5 лаб)
### CORS (3 лабы)
### CSRF (12 лаб)
### Clickjacking (5 лаб)
### HTTP Host Header (5 лаб)
### HTTP Request Smuggling (12 лаб)
### Information Disclosure (5 лаб)
### SSTI (5 лаб)
### Prototype Pollution (10 лаб)
### Web Cache Poisoning (11 лаб)
### WebSocket (3 лабы)
### NoSQL Injection
### File Upload
### Race Conditions

---

## 🏛️ Фундаментальные знания

### Сети
OSI / OSI 7 / TCP / UDP / IP / NAT / DNS / HTTP / WAF / SOCKET / PORTs / Tor / VPN / Web Cache / Анатомия веб-запроса

### Криптография
PKI-сертификаты / X.509 / Hash функции / SSL/TLS рукопожатия / E2EE / Асимметричное и симметричное шифрование

### Базы данных
SQL / XML / общее

### Языки программирования
основной мой яп SWIFT (написал 3+ приложения, + работа)
JavaScript (база + Secure + payloads) / Python / HTML / CSS / SQL

### Операционные системы
Linux архитектура / Windows архитектура / OS command / POINT RECON

### Браузеры
Browser DOM / BOM / JS теория

---

## 📱 Мобильная безопасность (MASTG / MASVS)

### ИТОГ: 100% покрытие iOS безопасности по MASVS

Решены ВСЕ тесты OWASP MASVS (L1, L2, L3/R).
Протестировано на реальном приложении - соц-сеть MeetWay (iOS).
Настроена защита + взломана собственная защита

### Статистика тестов по категориям MASVS

MASVS-AUTH (аутентификация, биометрия) — 5 тестов (L1, L2)
MASVS-CODE (SCA, ARC, канарейки) — 3 теста (L1, L2)
MASVS-CRYPTO (криптография, ключи, хеши) — 7 тестов (L1, L2)
MASVS-NETWORK (TLS, ATS, Pinning, сокеты) — 4 теста (L1, L2)
MASVS-PLATFORM (iOS платформа) — 23 теста (L1, L2)
MASVS-PRIVACY (Privacy Manifest) — 1 тест (P)
MASVS-RESILIENCE (реверс, jailbreak, анти-дебаг, обфускация) — 10+ тестов (R)
MASVS-STORAGE (логи, бэкапы, кеш) — 5 тестов (L1, L2)

Дополнительно: Android тестирование (базовое, L1)

**ВСЕГО МОБИЛЬНЫХ ТЕСТОВ / ЛАБОРАТОРНЫХ: 65+**

---

## 🛠️ Инструменты

### Динамический анализ (DAST / Runtime)
Frida / Objection / LLDB / radare2 / Burp Suite

### Статический анализ (SAST)
MobSF / Semgrep / strings / nm / otool

### Работа с устройством и файловой системой
palera1n (джейлбрейк iPhone 8 iOS 16.7.14) / keychain_dumper / iMazing / SSH

### Среда разработки и тестирования
Xcode / iOS simulator

---

## 🧠 Компетенции по iOS / Mobile AppSec

### Атаки / пентест (DAST)
Обход SSL Pinning (с джейлбрейком и без)
Перехват трафика реального iPhone (WiFi + Burp)
Перехват трафика iOS симулятора
Обход Jailbreak detection (код + runtime: Frida, radare2, LLDB, патчинг)
Обход Anti-debug detection
Keychain — извлечение данных
Дамп логов / утечки данных
Проверка приватного хранилища / бэкапов / кеша клавиатуры

### Защита (hardening)
Certificate Pinning / Jailbreak detection (код + runtime)
Обфускация / проверка целостности IPA
Отключение бэкапов для чувствительных данных
ARC / Stack Canaries / PIC
Удаление отладочных символов
Privacy Manifest / Secure Enclave

### Аудит кода (SAST / реверс)
Хардкод ключей / Insecure Random API
Слабые алгоритмы шифрования (ECB, устаревшие)
Слабые хеши (MD5, SHA1)
Проверка TLS / ATS
SCA + SBOM (Dependency-Track)

---

## 🔥 Ключевые достижения в мобильной безопасности

1. 100% покрытие MASVS (все тесты, все уровни)
2. Реальный проект — защита в рабочей соц-сети MeetWay
3. Сам взломал свою защиту → подтвердил эффективность
4. Освоил стек: Frida / LLDB / radare2 / MobSF / Semgrep / Burp / keychain_dumper / palera1n
5. Перехват трафика с реального iPhone и симулятора
6. Полный цикл: атака → защита → аудит → повторный аудит

---

## 📁 Структура репозитория (MASTGE и смежное)

MASTGE/
├── MASVS-AUTH/         (5 тестов)
├── MASVS-CODE/         (3 теста)
├── MASVS-CRYPTO/       (7 тестов)
├── MASVS-NETWORK/      (4 теста)
├── MASVS-PLATFORM/     (23 теста)
├── MASVS-PRIVACY/      (Privacy Manifest)
├── MASVS-RESILIENCE/   (10+ тестов)
├── MASVS-STORAGE/      (5 тестов)
├── android-testing/    (Android L1)
├── ios_testing_MeetWay/ (аудит соц-сети)
├── tools+methods/      (Frida, LLDB, MobSF, Semgrep, radare2, keychain_dumper, iMazing)
├── общая_теория_введение-кратко/
├── CI-CD / DevSecOps roadmap
└── assets/

Дополнительно:
- Аудит соц-сети MeetWay / план проверки
- Собеседования (Web & App Pentest / AppSec)
- Ссылки по инструментам и темам

---

## 🧪 CI-CD / DevSecOps

Начато изучение CI-CD.
В репозитории присутствует roadmap:
`общее_CI_CD-DevSecOps===roadmap.md`

---

> 📌 Весь репозиторий рекомендуется открывать в Obsidian — каждая статья будет в цвете и лучше структурирована, чем на GitHub.