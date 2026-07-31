# Инструменты анализа: SAST, DAST, IAST, RASP, SCA

> AppSec-руководство · раздел 9/22 · сформировано 31.07.2026
> Полное руководство: см. остальные файлы в папке SDLC


### 9.1 SAST – Static Application Security Testing

**SAST (статическое тестирование безопасности)** – анализ исходного кода БЕЗ его запуска. Сканер ищет паттерны уязвимостей в коде.
> **SAST** – Static Application Security Testing (статическое тестирование безопасности приложений)


**Как работает:** строит AST (Abstract Syntax Tree) / граф потоков данных и сопоставляет с правилами уязвимых паттернов (taint analysis, data flow analysis).
> **AST** – Abstract Syntax Tree (абстрактное синтаксическое дерево)


**Что находит:**
- SQL-инъекции, XSS, command injection
- Небезопасные вызовы функций (eval, exec)
- Хардкод секретов
- Небезопасное использование криптографии
- Утечки данных (taint: пользовательский ввод → опасная функция)
> **XSS** – Cross-Site Scripting (межсайтовый скриптинг)

> **SQL** – Structured Query Language (структурированный язык запросов)


**Плюсы:** быстро, дёшево, находит проблемы сразу в коде, покрывает весь код, не требует запущенного приложения.
**Минусы:** ложные срабатывания (false positives), не видит runtime-контекст и конфигурацию.

**Инструменты:** Semgrep, CodeQL (GitHub), SonarQube, Checkmarx, Fortify SCA, Veracode, PVS-Studio.
> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)


**Пример (Semgrep-правило для SQL-инъекции):**
```yaml
rules:
  - id: sql-injection
    patterns:
      - pattern: execute($QUERY + $USER_INPUT)
    message: Возможная SQL-инъекция
    languages: [python]
    severity: WARNING
```

### 9.2 DAST – Dynamic Application Security Testing

**DAST (динамическое тестирование безопасности)** – анализ ЗАПУЩЕННОГО приложения "снаружи", как чёрный ящик. Сканер отправляет HTTP-запросы и анализирует ответы.
> **HTTP** – HyperText Transfer Protocol (протокол передачи гипертекста)

> **DAST** – Dynamic Application Security Testing (динамическое тестирование безопасности приложений)


**Как работает:** автоматизированный сканер (или фаззер) подаёт вредоносные пейлоады в приложение через его интерфейсы (HTTP API, формы) и ищет аномалии.
> **API** – Application Programming Interface (программный интерфейс приложения)


**Что находит:**
- Инъекции (SQLi, XSS, SSTI, XXE) на работающем приложении
- Проблемы аутентификации/авторизации (IDOR, broken access control)
- Утечки информации в ответах
- Ошибки конфигурации
- CSRF, CORS-проблемы
> **IDOR** – Insecure Direct Object Reference (небезопасная прямая ссылка на объект)

> **SQLi** – SQL Injection (инъекция SQL)


**Плюсы:** видит реальное поведение приложения, меньше false positives по логике, не нужен исходный код.
**Минусы:** требует запущенного приложения и окружения, медленнее, покрывает только доступные пути.

**Инструменты:** OWASP ZAP, Burp Suite (Pro), Acunetix, AppScan, Nessus (частично).
> **ZAP** – Zed Attack Proxy (инструмент OWASP для тестирования безопасности веб-приложений)

> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


**Пример использования (ZAP headless):**
```bash
zap.sh -cmd -quickurl http://target:8080 -quickout report.html
```

### 9.3 IAST – Interactive Application Security Testing

**IAST (интерактивное тестирование безопасности)** – гибрид SAST и DAST: агент встраивается в приложение (обычно как агент в runtime/JVM) и анализирует выполнение в реальном времени во время обычного использования или автоматических тестов.
> **IAST** – Interactive Application Security Testing (интерактивное тестирование безопасности приложений)


**Как работает:** инструментация приложения (agent/sensor), мониторинг потоков данных в runtime, выявление уязвимостей на основе реального выполнения.

**Что находит:** те же классы, что SAST+DAST, но с точным подтверждением (реальное выполнение = меньше false positives). Видит точный путь данных от входа до опасной функции.

**Плюсы:** высокая точность, видит реальные потоки данных, работает с существующими тестами.
**Минусы:** нужна инструментация приложения, накладные расходы на runtime, покрытие зависит от используемых сценариев.

**Инструменты:** Contrast Security, Seeker (Synopsys), HCL AppScan IAST, Acunetix IAST.

### 9.4 RASP – Runtime Application Self-Protection

**RASP (самозащита приложения в рантайме)** – технология, встроенная в приложение (или его окружение), которая обнаруживает и блокирует атаки в реальном времени.
> **RASP** – Runtime Application Self-Protection (самозащита приложения во время выполнения)


**Как работает:** агент RASP интегрируется в приложение (библиотека/агент в runtime), анализирует поведение (вызовы функций, потоки данных) и при обнаружении атаки блокирует её, не дожидаясь исправления кода.

**Что делает:**
- Обнаруживает и блокирует инъекции (SQLi, XSS) в рантайме
- Блокирует эксплуатацию известных уязвимостей
- Ведёт подробный аудит атак
- Не требует изменения кода приложения

**Плюсы:** защита в реальном времени, работает даже при неисправленных уязвимостях, детальная телеметрия.
**Минусы:** производительность, ложные срабатывания, должен быть установлен в каждом экземпляре.

**Инструменты:** Hdiv, Contrast Protect, Imperva RASP, Sqreen (Datadog), open-appsec.

### 9.5 SCA – Software Composition Analysis

**SCA (анализ состава ПО)** – анализ сторонних компонентов и зависимостей (open-source библиотек) на наличие известных уязвимостей и лицензионных проблем.
> **ПО** – Software (программное обеспечение)


**Как работает:** сверяет используемые версии библиотек (из lock-файлов, package managers) с базами уязвимостей (CVE, NVD, GitHub Advisory, OSV).
> **OSV** – Open Source Vulnerabilities (открытая база уязвимостей открытого ПО (Google))

> **NVD** – National Vulnerability Database (национальная база данных уязвимостей (США, NIST))

> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))


**Что находит:**
- Известные CVE в зависимостях
- Устаревшие версии библиотек
- Лицензионные риски (GPL, AGPL в коммерческом продукте)
- Транзитивные зависимости (зависимости зависимостей)
- Вредоносные пакеты (malicious packages, typosquatting)

**Плюсы:** закрывает самый массовый класс уязвимостей (уязвимые компоненты ~ в 90% приложений), автоматизируется в CI.
**Минусы:** не видит собственный код, зависит от актуальности баз, шум от неиспользуемых уязвимых функций.

**Инструменты:** OWASP Dependency-Check, OWASP Dependency-Track, Snyk, Trivy, Grype, WhiteSource/Mend, GitHub Dependabot, Nexus IQ, JFrog Xray.

### Сравнительная таблица

| | SAST | DAST | IAST | RASP | SCA |
|--|------|------|------|------|-----|
| **Когда** | Код написан | Приложение запущено | Во время работы | Runtime | Сборка/CI |
| **Доступ** | Исходники | Снаружи (чёрный ящик) | Внутри (агент) | Внутри (агент) | Зависимости |
| **False positives** | Много | Средне | Мало | Средне | Низко |
| **Скорость** | Быстро | Медленно | Средне | Real-time | Быстро |
| **Этап** | Shift-left | Pre-release | Testing | Production | Build |


---

## 📎 Пример: отчёт SAST-сканера (Semgrep) – как это выглядит

> Фрагмент реального JSON-вывода Semgrep (упрощён).

```json
{
  "results": [
    {
      "check_id": "python.lang.security.audit.eval-usage.eval-usage",
      "path": "app/views/search.py",
      "start": {"line": 42, "col": 12},
      "end": {"line": 42, "col": 25},
      "extra": {
        "message": "Использование eval() с пользовательским вводом - риск RCE",
        "severity": "ERROR",
        "metadata": {
          "cwe": ["CWE-95"],
          "owasp": ["A03:2021 - Injection"]
        }
      }
    },
    {
      "check_id": "python.lang.security.audit.hardcoded-password",
      "path": "config/settings.py",
      "start": {"line": 10, "col": 20},
      "extra": {
        "message": "Хардкод пароля в коде",
        "severity": "WARNING",
        "metadata": {"cwe": ["CWE-798"]}
      }
    }
  ],
  "errors": []
}
```

**Как читать отчёт:**
- `check_id` – какое правило сработало
- `path:line` – где именно в коде
- `severity` – ERROR (критично) / WARNING
- `cwe` – тип слабости, `owasp` – категория OWASP Top 10

---

## 📎 Пример: правила Semgrep (свои правила для команды)

```yaml
# .semgrep/rules/security.yml
rules:
  # SQL-инъекция: конкатенация строк в запросе
  - id: sql-injection-concat
    patterns:
      - pattern: execute($QUERY + $USER_INPUT)
      - pattern: cursor.execute(f"...{ $USER_INPUT }...")
    message: Возможная SQL-инъекция, используй параметризованные запросы
    languages: [python]
    severity: ERROR
    metadata:
      cwe: ["CWE-89"]

  # Запрет MD5/SHA1 для паролей
  - id: weak-hash-password
    pattern: hashlib.$FUNC($PASS)
    metavariable-regex:
      metavariable: $FUNC
      regex: (md5|sha1)
    message: Слабый хэш для пароля, используй bcrypt/argon2
    languages: [python]
    severity: ERROR
    metadata:
      cwe: ["CWE-327"]
```

---

## 📎 Пример: отчёт SCA (Trivy) – фрагмент

```bash
$ trivy image myapp:1.0

Total: 14 (UNKNOWN: 0, LOW: 6, MEDIUM: 5, HIGH: 2, CRITICAL: 1)

┌──────────────────┬──────────────┬──────────┬──────────────┬─────────────┐
│    Library       │ Vulnerability│ Severity │  Installed   │  Fixed      │
├──────────────────┼──────────────┼──────────┼──────────────┼─────────────┤
│ log4j-core       │ CVE-2021-44228│ CRITICAL │ 2.14.1       │ 2.17.0      │
│ openssl          │ CVE-2024-XXXX│ HIGH     │ 3.0.7        │ 3.0.13      │
│ libxml2          │ CVE-2023-XXXX│ HIGH     │ 2.9.13       │ 2.9.14      │
└──────────────────┴──────────────┴──────────┴──────────────┴─────────────┘
```

**Что делать с отчётом:**
1. CRITICAL/HIGH: обновить зависимость (Fixed version) или найти компенсирующие меры
2. MEDIUM/LOW: запланировать обновление
3. Настроить `--exit-code 1` для блокировки сборки при Critical

---

## 📎 Пример: DAST-отчёт (OWASP ZAP) – фрагмент

```html
<!-- Фрагмент отчёта ZAP: алерты -->
Alert: SQL Injection (SQLi)
  Risk: High
  URL: https://staging.example.com/search?q=test
  Description: Параметр q вставляется в SQL-запрос без параметризации
  Evidence: ' OR '1'='1
  CWE: CWE-89
  Recommendation: Использовать prepared statements

Alert: Missing Content-Security-Policy header
  Risk: Low
  URL: https://staging.example.com/
  CWE: CWE-693
  Recommendation: Добавить заголовок CSP
```

**Порядок работы с DAST:**
1. Сканирование staging после каждого деплоя (baseline scan)
2. Full scan перед релизом
3. Ручная проверка найденных High/Critical (исключить false positives)
4. Завести задачи в трекер с приоритетом
