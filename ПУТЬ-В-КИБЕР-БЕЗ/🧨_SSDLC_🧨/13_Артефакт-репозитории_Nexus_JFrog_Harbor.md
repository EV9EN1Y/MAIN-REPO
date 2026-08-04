# Артефакт-репозитории: Nexus, JFrog Artifactory, Harbor

> Полное руководство: см. остальные файлы в папке SDLC


**Артефакт-репозиторий** – централизованное хранилище бинарных артефактов (библиотек, образов, пакетов) с контролем версий, правами доступа и политиками безопасности.

### 13.1 Sonatype Nexus Repository

**Nexus** – артефакт-репозиторий от Sonatype. Хранит: Maven, npm, PyPI, Docker, NuGet и др.

**Возможности безопасности:**
- **Nexus IQ Server** (коммерческий) – политики безопасности компонентов
- **Vulnerability Age / Quarantine** – карантин уязвимых компонентов
- Интеграция с SCA: блокировка сборки при нарушении политики
- **Firewall** (Nexus Firewall) – блокировка вредоносных компонентов до загрузки
> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)


**Nexus как часть SSDLC:** разработчик не может скачать/использовать уязвимый компонент – он блокируется на уровне прокси-репозитория.
> **SSDLC** – Secure Software Development Life Cycle (безопасный жизненный цикл разработки программного обеспечения)


### 13.2 JFrog Artifactory

**Artifactory** – артефакт-репозиторий от JFrog (Universal Repository Manager).

**Возможности безопасности:**
- **JFrog Xray** – SCA-сканирование артефактов, зависимостей и Docker-образов
- Анализ: CVE, лицензии, вредоносные пакеты
- **Watch + Policies** – политики безопасности (block on critical CVE)
- Интеграция с CI/CD (Jenkins, GitHub Actions)
- **SBOM** – генерация и проверка SBOM (CycloneDX/SPDX)
- **Impact Analysis** – при новой CVE показывает, какие артефакты затронуты
> **SPDX** – Software Package Data Exchange (формат обмена данными о пакетах ПО (Linux Foundation))

> **CI/CD** – Continuous Integration / Continuous Delivery (непрерывная интеграция / непрерывная поставка)

> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))

> **SBOM** – Software Bill of Materials (ведомость (спецификация) состава программного обеспечения)


### 13.3 Harbor

**Harbor** – open-source registry для контейнеров (CNCF project). Хранит Docker/OCI-образы.
> **CNCF** – Cloud Native Computing Foundation (фонд облачных (нативных) вычислений)

> **OCI** – Open Container Initiative (инициатива открытых контейнеров)


**Возможности безопасности:**
- **Trivy integration** – сканирование образов на CVE (встроенный Trivy)
- **Политики безопасности** – блокировка push уязвимых образов
- **Signing (Cosign/Notation)** – подпись образов, проверка подлинности
- **RBAC** – управление доступом к образам
- **Replication** – репликация между registry
- **Retention policies** – очистка старых образов
- **SBOM** – генерация SBOM для образов
> **RBAC** – Role-Based Access Control (управление доступом на основе ролей)


**Пример (Harbor + Trivy):**
```bash
# сканирование образа в Harbor через API
curl -X POST "https://harbor.example.com/api/v2.0/projects/myproject/repositories/myimage/artifacts/latest/scan"

# локально Trivy
trivy image harbor.example.com/myproject/myimage:latest
```

### Сравнение

| | Nexus | JFrog Artifactory | Harbor |
|--|-------|-------------------|--------|
| **Тип** | Universal repo | Universal repo | Container registry |
| **Фокус** | Компоненты/пакеты | Компоненты + артефакты | Docker/OCI образы |
| **SCA** | Nexus IQ | Xray | Trivy |
| **SBOM** | Да | Да | Да |
| **Лицензии** | OSS/Pro | OSS/Pro/Enterprise | Open-source |
| **Когда выбрать** | Java/Maven экосистема | Универсальное, enterprise | Только контейнеры |


---

## 📎 Пример: политика безопасности в Nexus (Nexus IQ)

> Пример policy-файла Nexus IQ (YAML) – блокировка уязвимых компонентов.

```yaml
# nexus-iq-policy.yaml
policy:
  - id: block-critical-cve
    name: Блокировка Critical CVE
    description: Запрещает использование компонентов с критичными уязвимостями
    constraints:
      - id: critical-cve-constraint
        conditions:
          - type: vulnerability
            params:
              severity: critical
        actions:
          - type: fail-build   # сборка падает
          - type: email-notification
            params:
              to: security@company.com
  - id: license-risk
    name: Запрет копилефт-лицензий
    constraints:
      - conditions:
          - type: license
            params:
              licenses: [GPL-3.0, AGPL-3.0]
        actions:
          - type: warn
```

**Как работает:** разработчик пытается добавить зависимость с Critical CVE → Nexus IQ блокирует (build fail) или ставит в карантин, письмо уходит security-команде.

---

## 📎 Пример: политика JFrog Xray (Watch + Policy)

```yaml
# Xray policy (пример)
policies:
  - name: "Block Critical in Production"
    type: security
    rules:
      - name: "Critical CVE"
        criteria:
          min_severity: critical
          cvss_score: 9.0
        actions:
          - block_download
          - fail_build
          - notify:
              recipients: [security@company.com]
```

**Команды Xray:**
```bash
# сканирование артефакта через API
curl -X POST "https://artifactory.example.com/xray/api/v1/scanArtifact" \
  -H "Content-Type: application/json" \
  -d '{"componentID": "docker://registry.example.com/app:1.0.0"}'
```

---

## 📎 Пример: политика Harbor (допуск образов)

```yaml
# Harbor - пример политики (UI/API)
project: production
security:
  vulnerability_scanning: enabled
  scan_on_push: true
  block_on_critical: true        # блокировать push при Critical
  require_signature: true        # требовать подпись (Cosign)
  prevention:
    severity: critical
    action: block
```

**Пример пайплайна с Harbor + Cosign (подпись образа):**
```bash
# 1. Собрать образ
docker build -t harbor.example.com/prod/app:1.0.0 .

# 2. Сканирование (автоматически при push, если scan_on_push)
docker push harbor.example.com/prod/app:1.0.0

# 3. Подписать образ (Cosign)
cosign sign harbor.example.com/prod/app:1.0.0

# 4. Проверить подпись перед деплоем
cosign verify harbor.example.com/prod/app:1.0.0

# 5. Сканирование локально до пуша (Trivy)
trivy image harbor.example.com/prod/app:1.0.0 --severity CRITICAL --exit-code 1
```

---

## 📎 Сравнение: что где хранить (шпаргалка)

| Артефакт | Nexus | JFrog | Harbor |
|----------|-------|-------|--------|
| Java/Maven библиотеки | ✅ лучший | ✅ | - |
| npm/PyPI пакеты | ✅ | ✅ | - |
| Docker-образы | ✅ | ✅ | ✅ лучший |
| SBOM | ✅ (IQ) | ✅ (Xray) | ✅ (Trivy) |
| Подпись артефактов | ✅ | ✅ | ✅ (Cosign) |
| Блокировка по CVE | ✅ | ✅ | ✅ |
