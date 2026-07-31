# CVE, CWE, CVSS – база уязвимостей

> Полное руководство: см. остальные файлы в папке SDLC


### 11.1 CVE – Common Vulnerabilities and Exposures

**CVE** – это реестр (словарь) публично известных уязвимостей и проблем безопасности. Каждая уязвимость получает уникальный идентификатор.
> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))


**Формат идентификатора:** `CVE-YYYY-NNNN`
- `CVE` – префикс
- `YYYY` – год присвоения
- `NNNN` – порядковый номер

**Примеры:** CVE-2021-44228 (Log4Shell), CVE-2017-0144 (EternalBlue), CVE-2024-3400 (PAN-OS)
> **OS** – Operating System (операционная система)


**Кто присваивает:** CVE Program (MITRE) через CNA (CVE Numbering Authorities) – организации, уполномоченные присваивать CVE (Microsoft, Google, Apache, Oracle и др.).
> **MITRE** – (некоммерческая организация, управляющая реестром CVE) (организация-оператор реестра CVE)


**Поля записи CVE:**
- **Description** – описание уязвимости
- **References** – ссылки (advisory производителя, PoC)
- **Date** – дата создания записи
- **CWE** – ссылка на категорию слабостей
> **PoC** – Proof of Concept (доказательство концепции (работоспособности эксплойта))

> **CWE** – Common Weakness Enumeration (общий перечень слабостей (типов уязвимостей))


### 11.2 CWE – Common Weakness Enumeration

**CWE** – таксономия (каталог) типов слабостей (weaknesses) в ПО. Если CVE – конкретные экземпляры, то CWE – классы проблем.
> **ПО** – Software (программное обеспечение)


**Примеры:**
- CWE-89: SQL Injection
- CWE-79: Cross-site Scripting (XSS)
- CWE-287: Improper Authentication
- CWE-502: Deserialization of Untrusted Data
- CWE-798: Use of Hard-coded Credentials
> **XSS** – Cross-Site Scripting (межсайтовый скриптинг)

> **SQL** – Structured Query Language (структурированный язык запросов)


**CWE Top 25** – ежегодный список самых опасных слабостей (по данным NIST/OWASP).
> **NIST** – National Institute of Standards and Technology (Национальный институт стандартов и технологий (США))

> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


### 11.3 CVSS – Common Vulnerability Scoring System

**CVSS** – стандартная система оценки severity (критичности) уязвимости. Оценка от 0 до 10.
> **CVSS** – Common Vulnerability Scoring System (общая система оценки критичности уязвимостей (0-10))


**Версии:** CVSS v2, v3.x (актуальна), v4.0 (вышла в 2023, постепенно внедряется).

**Группы метрик CVSS v3:**
1. **Base (базовые)** – свойства самой уязвимости (не меняются)
   - **AV – Attack Vector** (сетевой/локальный/физический)
   - **AC – Attack Complexity** (низкая/высокая)
   - **PR – Privileges Required** (нет/низкие/высокие)
   - **UI – User Interaction** (нет/требуется)
   - **S – Scope** (неизменный/изменённый)
   - **C/I/A – Confidentiality/Integrity/Availability Impact** (none/low/high)
2. **Temporal (временные)** – меняются со временем (зрелость эксплойта, наличие фикса)
3. **Environmental (окружение)** – специфика конкретной среды

**Шкала severity:**
| Оценка | Уровень |
|--------|---------|
| 0.0 | None |
| 0.1–3.9 | Low |
| 4.0–6.9 | Medium |
| 7.0–8.9 | High |
| 9.0–10.0 | Critical |

**Ограничения CVSS:** оценка не учитывает бизнес-контекст. Критичный CVSS 9.8 может быть неважен для системы, не подключённой к сети. Поэтому CVSS дополняют контекстной приоритизацией (EPSS, риск-оценка).
> **EPSS** – Exploit Prediction Scoring System (система прогнозирования вероятности эксплуатации уязвимостей)


**EPSS (Exploit Prediction Scoring System)** – модель (FIRST) оценивает вероятность эксплуатации уязвимости в ближайшие 30 дней (0–100%). Помогает приоритизировать: патчить то, что реально эксплуатируется.


---

## 📎 Пример: запись CVE в NVD (как это выглядит)

> Пример реальной структуры записи CVE (NVD API v2.0, сокращено).
> Взят реальный формат на примере CVE-2021-44228 (Log4Shell).

```json
{
  "id": "CVE-2021-44228",
  "sourceIdentifier": "security@apache.org",
  "published": "2021-12-10T10:15:00.000",
  "lastModified": "2023-02-24T18:00:00.000",
  "vulnStatus": "Analyzed",
  "descriptions": [
    {
      "lang": "en",
      "value": "Apache Log4j2 2.0-beta9 through 2.15.0 (excluding 2.12.2, 2.12.3) does not protect against uncontrolled recursion from self-referential lookups. This allows an attacker to control log messages to execute arbitrary code loaded from LDAP servers."
    }
  ],
  "metrics": {
    "cvssMetricV31": [{
      "source": "nvd@nist.gov",
      "type": "Primary",
      "cvssData": {
        "version": "3.1",
        "vectorString": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H",
        "attackVector": "NETWORK",
        "attackComplexity": "LOW",
        "privilegesRequired": "NONE",
        "userInteraction": "NONE",
        "scope": "CHANGED",
        "confidentialityImpact": "HIGH",
        "integrityImpact": "HIGH",
        "availabilityImpact": "HIGH",
        "baseScore": 10.0,
        "baseSeverity": "CRITICAL"
      }
    }]
  },
  "weaknesses": [{
    "source": "nvd@nist.gov",
    "type": "Primary",
    "description": [{"value": "CWE-502"}]
  }],
  "configurations": [{
    "nodes": [{
      "operator": "OR",
      "cpeMatch": [{
        "vulnerable": true,
        "criteria": "cpe:2.3:a:apache:log4j:*:*:*:*:*:*:*:*",
        "versionStartIncluding": "2.0-beta9",
        "versionEndExcluding": "2.15.0"
      }]
    }]
  }],
  "references": [
    {"url": "https://logging.apache.org/log4j/2.x/security.html"},
    {"url": "https://nvd.nist.gov/vuln/detail/CVE-2021-44228"}
  ]
}
```

**Как читать:**
- `id` – идентификатор CVE
- `descriptions` – описание уязвимости
- `metrics.cvssMetricV31` – оценка CVSS: вектор + базовый балл (10.0 = Critical)
- `weaknesses` – CWE-категория (CWE-502 – небезопасная десериализация)
- `configurations` – какие версии уязвимы (CPE)
- `references` – ссылки на advisory

---

## 📎 Как разбирать CVSS-вектор (шпаргалка)

Вектор: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` = 10.0

| Метрика | Значение | Расшифровка |
|---------|----------|-------------|
| AV:N | Attack Vector: Network | Атака по сети (извне) |
| AC:L | Attack Complexity: Low | Нет сложных условий |
| PR:N | Privileges Required: None | Без прав |
| UI:N | User Interaction: None | Без участия пользователя |
| S:C | Scope: Changed | Влияет за пределами компонента |
| C:H | Confidentiality: High | Полное раскрытие |
| I:H | Integrity: High | Полное изменение |
| A:H | Availability: High | Полный отказ |

---

## 📎 Пример: как искать CVE через NVD API

```bash
# Поиск по ключевому слову
curl "https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=log4j"

# Поиск по CPE (конкретный продукт)
curl "https://services.nvd.nist.gov/rest/json/cves/2.0?cpeName=cpe:2.3:a:apache:log4j:*"

# Поиск по дате (новые за неделю)
curl "https://services.nvd.nist.gov/rest/json/cves/2.0?pubStartDate=2026-07-24T00:00:00.000&pubEndDate=2026-07-31T00:00:00.000"

# БДУ ФСТЭК (российская база)
# https://bdu.fstec.ru/
```
