# SBOM – Software Bill of Materials

> Полное руководство: см. остальные файлы в папке SDLC


**SBOM (Software Bill of Materials)** – "ведомость состава ПО": формализованный список всех компонентов, библиотек и зависимостей, из которых собрано приложение. Аналог спецификации материалов в производстве (BOM в manufacturing).
> **ПО** – Software (программное обеспечение)

> **SBOM** – Software Bill of Materials (ведомость (спецификация) состава программного обеспечения)


**Что содержит SBOM:**
- Список всех компонентов (библиотек, пакетов)
- Версии каждого компонента
- Лицензии
- Происхождение (supplier, репозиторий)
- Хэши/контрольные суммы
- Зависимости между компонентами (включая транзитивные)

**Зачем нужен SBOM:**
- **Реагирование на CVE:** когда вышла уязвимость (например, Log4Shell) – по SBOM за минуты понять, затронут ли твой продукт
- **Управление цепочкой поставок:** знать, из чего собран каждый артефакт
- **Соответствие требованиям:** США (Executive Order 14028, CISA), ЕС (Cyber Resilience Act), регуляторы требуют SBOM
- **Аудит и due diligence**
> **CISA** – Cybersecurity and Infrastructure Security Agency (Агентство по кибербезопасности и защите инфраструктуры (США))

> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))


**Форматы SBOM (стандарты):**
- **SPDX (Software Package Data Exchange)** – формат от Linux Foundation, ISO/IEC 5962:2021
- **CycloneDX** – формат от OWASP, лёгкий, популярен для безопасности
- **SWID (Software Identification)** – ISO/IEC 19770-2, от NIST
> **ISO/IEC** – International Organization for Standardization / International Electrotechnical Commission (Международная организация по стандартизации / Международная электротехническая комиссия)

> **SWID** – Software Identification (идентификация программного обеспечения (ISO/IEC 19770-2))

> **SPDX** – Software Package Data Exchange (формат обмена данными о пакетах ПО (Linux Foundation))

> **NIST** – National Institute of Standards and Technology (Национальный институт стандартов и технологий (США))

> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


**Пример CycloneDX SBOM (фрагмент JSON):**
```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "components": [
    {
      "type": "library",
      "name": "log4j-core",
      "version": "2.14.1",
      "purl": "pkg:maven/org.apache.logging.log4j/log4j-core@2.14.1"
    }
  ]
}
```

**Как генерировать SBOM:**
- **Trivy:** `trivy image --format cyclonedx --output sbom.json myimage:latest`
- **Syft:** `syft myimage:latest -o spdx-json`
- **CycloneDX CLI / Maven plugin / Gradle plugin**
- **JFrog:** генерация SBOM из артефактов
- **GitHub:** Dependency Graph + SBOM export (Insights → Dependency Graph → Export SBOM)


---

## 📎 Пример: настоящий SBOM-файл (CycloneDX JSON)

> Фрагмент SBOM приложения, сгенерированный Trivy/Syft.

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "serialNumber": "urn:uuid:3e671687-395b-41f5-a30f-a58921a69b79",
  "version": 1,
  "metadata": {
    "timestamp": "2026-07-31T12:00:00Z",
    "tools": [{"vendor": "anchore", "name": "syft", "version": "0.98.0"}],
    "component": {
      "type": "application",
      "name": "meetway-api",
      "version": "1.0.0",
      "purl": "pkg:docker/registry.example.com/meetway-api@1.0.0"
    }
  },
  "components": [
    {
      "type": "library",
      "name": "log4j-core",
      "version": "2.17.1",
      "purl": "pkg:maven/org.apache.logging.log4j/log4j-core@2.17.1",
      "licenses": [{"license": {"id": "Apache-2.0"}}],
      "hashes": [{"alg": "SHA-256", "content": "f2a1c..."}],
      "externalReferences": [
        {"type": "vcs", "url": "https://github.com/apache/logging-log4j2"}
      ]
    },
    {
      "type": "library",
      "name": "fastapi",
      "version": "0.110.0",
      "purl": "pkg:pypi/fastapi@0.110.0",
      "licenses": [{"license": {"id": "MIT"}}]
    },
    {
      "type": "library",
      "name": "bcrypt",
      "version": "4.1.2",
      "purl": "pkg:pypi/bcrypt@4.1.2"
    }
  ],
  "dependencies": [
    {"ref": "pkg:docker/registry.example.com/meetway-api@1.0.0",
     "dependsOn": ["pkg:maven/org.apache.logging.log4j/log4j-core@2.17.1"]}
  ]
}
```

**Как читать:**
- `metadata.component` – что за приложение
- `components` – список зависимостей (имя, версия, purl, лицензия)
- `dependencies` – связи (транзитивные)
- `purl` – уникальный идентификатор пакета (для поиска CVE)

---

## 📎 Как сгенерировать SBOM (команды)

```bash
# Syft (образ -> SBOM)
syft registry.example.com/meetway-api:1.0.0 -o cyclonedx-json > sbom.json
syft registry.example.com/meetway-api:1.0.0 -o spdx-json > sbom.spdx.json

# Trivy
trivy image --format cyclonedx --output sbom.json registry.example.com/meetway-api:1.0.0

# GitHub: репозиторий -> Insights -> Dependency graph -> Export SBOM (SPDX)

# Maven (CycloneDX plugin)
mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom
```

---

## 📎 Как использовать SBOM при новой CVE (пример)

**Ситуация: вышла CVE-2026-XXXXX в библиотеке X.**

1. Ищем библиотеку в SBOM:
```bash
grep -i "log4j" sbom.json   # или jq
jq '.components[] | select(.name | test("log4j"))' sbom.json
```

2. Проверяем версию: если версия в SBOM < исправленной - затронуты
3. Находим, в каких артефактах/сервисах используется:
```bash
for f in sbom-*.json; do echo "$f: $(jq '.components[] | select(.name=="log4j-core") | .version' $f)"; done
```
4. План: обновить версию, пересобрать, пересканировать
