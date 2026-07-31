# Shift Left – смещение безопасности влево

> Полное руководство: см. остальные файлы в папке SDLC


**Shift Left** – подход, при котором безопасность смещается на ранние этапы жизненного цикла разработки ("влево" на диаграмме SDLC). Вместо проверки безопасности в конце (пентест перед релизом) – проверки встроены с самого начала.
> **SDLC** – Software Development Life Cycle (жизненный цикл разработки программного обеспечения)


**Суть:** найти уязвимость как можно раньше, когда исправление стоит копейки, а не миллионы.

**Стоимость исправления бага (примерная):**
- Требования/дизайн: x1
- Разработка: x6.5
- Тестирование: x15
- Продакшн: x100

**Что значит "влево" на практике:**
- Требования → security-требования в user stories
- Дизайн → threat modeling, review архитектуры
- Код → SAST в IDE (pre-commit), code review, secure coding стандарты
- Сборка → SCA зависимостей, проверка SBOM
- Тестирование → автоматизированные security-тесты в CI
- Деплой → сканирование контейнеров, IaC-сканирование
> **IaC** – Infrastructure as Code (инфраструктура как код)

> **IDE** – Integrated Development Environment (интегрированная среда разработки)

> **SBOM** – Software Bill of Materials (ведомость (спецификация) состава программного обеспечения)

> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)

> **SAST** – Static Application Security Testing (статическое тестирование безопасности приложений)


**Инструменты shift left:**
- SAST: Semgrep, SonarQube, CodeQL, Fortify
- SCA: OWASP Dependency-Check, Snyk, Trivy
- IaC: Checkov, tfsec, KICS
- Secrets scanning: gitleaks, trufflehog, detect-secrets
- IDE-плагины: SonarLint, Semgrep IDE, Snyk IDE
> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


**Shift Left vs Shift Everywhere:** современный подход – не только "влево", но и "везде": безопасность на всех этапах, включая эксплуатацию (RASP, мониторинг).
> **RASP** – Runtime Application Self-Protection (самозащита приложения во время выполнения)


---

## 📎 Пример: DevSecOps пайплайн (CI/CD) с Shift Left

> Пример `.gitlab-ci.yml` / GitHub Actions – как встроить безопасность в пайплайн.

```yaml
# .gitlab-ci.yml (пример DevSecOps пайплайна)
stages:
  - security-static   # сдвиг влево: анализ ДО сборки
  - build
  - security-deps     # SCA на этапе сборки
  - test
  - security-dynamic  # DAST на staging
  - deploy

# 1. SAST - статический анализ кода (сдвиг влево, до сборки)
sast:
  stage: security-static
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto --json -o semgrep.json .
    - semgrep --config=auto --error --severity=ERROR .
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# 2. Поиск секретов (gitleaks)
secret-detection:
  stage: security-static
  image: zricethezav/gitleaks
  script:
    - gitleaks detect --source . --report-format json --report-path gitleaks.json
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# 3. SCA - анализ зависимостей (после сборки)
sca:
  stage: security-deps
  script:
    - trivy fs --severity CRITICAL,HIGH --exit-code 1 .
    - trivy fs --format cyclonedx --output sbom.json .
  artifacts:
    paths: [sbom.json]

# 4. DAST - динамический анализ (на staging)
dast:
  stage: security-dynamic
  script:
    - docker run -t owasp/zap2docker-stable zap-baseline.py -t https://staging.example.com
  allow_failure: false

# 5. Сканирование Docker-образа перед деплоем
container-scan:
  stage: security-dynamic
  script:
    - trivy image --severity CRITICAL --exit-code 1 registry.example.com/app:$CI_COMMIT_SHA
```

**Как работает Shift Left в этом пайплайне:**
1. SAST и gitleaks запускаются на **каждом merge request** – уязвимость ловится до мержа
2. SCA проверяет зависимости при каждой сборке
3. DAST бьёт по staging перед деплоем
4. Образ сканируется перед выкаткой в прод

---

## 📎 Схема Shift Left (текстовая)

```
        Сдвиг влево: проверки раньше = дешевле фикс
        <-------------------------------------------------
        
Требования   Дизайн     Код        Сборка     Тесты     Деплой    Прод
   |           |          |          |          |         |         |
   v           v          v          v          v         v         v
[SecReq]  [ThreatModel] [SAST]    [SCA]      [DAST]   [ConfigScan] [RASP]
                         [Secrets] [SBOM]     [Pentest]             [SIEM]
                         [Review]
```

**Стоимость исправления (множитель):**
| Этап обнаружения | Относительная стоимость |
|------------------|-------------------------|
| Требования/дизайн | x1 |
| Разработка | x6.5 |
| Тестирование | x15 |
| Продакшн | x100 |
