# SSDLC – безопасный жизненный цикл разработки

> Полное руководство: см. остальные файлы в папке SDLC


**SSDLC (Secure Software Development Life Cycle)** – это SDLC, в который безопасность встроена в каждую фазу, а не добавлена в конце.
> **SSDLC** – Secure Software Development Life Cycle (безопасный жизненный цикл разработки программного обеспечения)

> **SDLC** – Software Development Life Cycle (жизненный цикл разработки программного обеспечения)


**Ключевое отличие от SDLC:** в традиционном SDLC безопасность появляется только на этапе тестирования ("проверить и исправить"). SSDLC задаёт вопрос на старте: "Как сделать это безопасно?" – до написания первой строки кода.

**IBM выделяет 9 фаз SSDLC:**
1. **Requirements (требования)** – обсуждение security-требований вместе с функциональными
2. **Analysis (анализ)** – анализ угроз и рисков
3. **Planning (планирование)** – план внедрения мер безопасности
4. **Design (дизайн)** – secure architecture, threat modeling
5. **Development (разработка)** – secure coding, валидация входных данных, стандарты аутентификации
6. **Documentation (документация)** – документирование мер безопасности, модели угроз
7. **Testing (тестирование)** – непрерывное тестирование, автоматизированные code review
8. **Deployment (внедрение)** – безопасная конфигурация, проверка окружения
9. **Maintenance (поддержка)** – мониторинг, патчинг, реагирование на инциденты
> **IBM** – International Business Machines (корпорация IBM)


**Как внедрить SSDLC в существующий процесс:**
1. Назначить ответственного за безопасность (AppSec-инженера или security champion)
2. Встроить security-требования в user stories и acceptance criteria
3. Внедрить threat modeling на этапе дизайна
4. Подключить SAST в IDE и CI (pre-commit / merge request)
5. Подключить SCA для анализа зависимостей
6. Автоматизировать DAST для staging-окружений
7. Внедрить процесс реагирования на уязвимости (vulnerability management)
8. Обучать разработчиков (security awareness, secure coding)
> **IDE** – Integrated Development Environment (интегрированная среда разработки)

> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)

> **DAST** – Dynamic Application Security Testing (динамическое тестирование безопасности приложений)

> **SAST** – Static Application Security Testing (статическое тестирование безопасности приложений)

> **AppSec** – Application Security (безопасность приложений)


**Стандарты и модели зрелости SSDLC:**
- **OWASP SAMM (Software Assurance Maturity Model)** – модель зрелости: 5 бизнес-функций (Governance, Design, Implementation, Verification, Operations), 15 практик, 3 уровня зрелости
- **BSIMM (Building Security In Maturity Model)** – 12 практик, 4 домена (Governance, Intelligence, SSDL Touchpoints, Deployment)
- **MS-SDL (Microsoft Security Development Lifecycle)** – 14 практик от Microsoft
- **NIST SSDF (Secure Software Development Framework, SP 800-218)** – 4 группы практик: Prepare, Protect, Produce, Respond
> **NIST** – National Institute of Standards and Technology (Национальный институт стандартов и технологий (США))

> **SSDF** – Secure Software Development Framework (структура (фреймворк) безопасной разработки ПО, NIST SP 800-218)

> **BSIMM** – Building Security In Maturity Model (модель зрелости встроенной безопасности)

> **SAMM** – Software Assurance Maturity Model (модель зрелости обеспечения гарантий безопасности ПО)

> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


---

## 📎 Пример: план внедрения SSDLC в реальный проект (шаблон)

> Документ: `SSDLC_Implementation_Plan.md` – план внедрения безопасной разработки
> Проект: любое приложение с командой 5-10 разработчиков.

```markdown
# План внедрения SSDLC в проект

## Цель
Встроить безопасность во все этапы разработки без остановки поставки.

## Фаза 0. Подготовка (неделя 1)
- [ ] Назначить AppSec-инженера / security champion
- [ ] Провести аудит текущего процесса (что уже есть: code review? CI?)
- [ ] Обучить команду: 2 воркшопа по secure coding (2 часа)

## Фаза 1. Требования (неделя 2)
- [ ] Создать шаблон security-требований (чек-лист)
- [ ] Встроить в user story поле "Security Acceptance Criteria"
- [ ] Проверить регуляторные требования (ФЗ-152, ГОСТ, PCI DSS)

## Фаза 2. Дизайн (неделя 2-3)
- [ ] Внедрить threat modeling для каждого нового эпика (STRIDE)
- [ ] Создать шаблон DFD-диаграммы
- [ ] Проверять архитектуру на security review

## Фаза 3. Разработка (неделя 3+)
- [ ] Подключить SAST (Semgrep) в IDE каждому разработчику
- [ ] Добавить SAST в CI на каждый merge request
- [ ] Подключить gitleaks (поиск секретов) в pre-commit
- [ ] Ввести обязательный code review (минимум 1 reviewer)

## Фаза 4. Сборка (неделя 4+)
- [ ] Подключить SCA (Trivy/Dependency-Check) в CI
- [ ] Настроить генерацию SBOM на каждую сборку
- [ ] Настроить блокировку сборки при Critical CVE

## Фаза 5. Тестирование (неделя 5+)
- [ ] Подключить DAST (OWASP ZAP) на staging
- [ ] Планировать пентест перед каждым крупным релизом
- [ ] Завести процесс приоритизации уязвимостей (CVSS + EPSS)

## Фаза 6. Деплой (неделя 6+)
- [ ] Чек-лист безопасного деплоя (конфигурация, заголовки, TLS)
- [ ] Сканирование окружения перед выкаткой

## Фаза 7. Эксплуатация
- [ ] Мониторинг уязвимостей (подписка на CVE)
- [ ] Регламент реагирования на инциденты (runbook)
- [ ] Периодические ревью модели угроз (раз в квартал)

## Метрики успеха
- SAST/SCA подключены в CI (да/нет)
- % задач с security acceptance criteria = 100%
- Critical CVE закрываются < 48 часов
- Время на security-проверки не замедляет релиз > 20%
```

---

## 📎 Пример: чек-лист безопасности для каждого этапа (шпаргалка)

```markdown
# Security Checklist (для каждой фазы)

## Требования
- [ ] Определены типы данных (ПДн? платёжные?)
- [ ] Регуляторные требования выявлены
- [ ] Security acceptance criteria в user story

## Дизайн
- [ ] Threat modeling проведён (STRIDE)
- [ ] DFD построена, границы доверия отмечены
- [ ] Выбор технологий согласован с AppSec

## Разработка
- [ ] SAST запущен, критичных нет
- [ ] Секреты не в коде (gitleaks чист)
- [ ] Code review пройден

## Сборка
- [ ] SCA чист (нет Critical CVE)
- [ ] SBOM сгенерирован
- [ ] Образы подписаны

## Тестирование
- [ ] DAST на staging: Critical/High = 0
- [ ] Пентест (перед крупным релизом)
- [ ] Уязвимости приоритизированы и запланированы

## Деплой
- [ ] DEBUG выключен, TLS 1.2+, заголовки безопасности
- [ ] Окружение просканировано
- [ ] Роллбэк проверен

## Эксплуатация
- [ ] Мониторинг и алерты работают
- [ ] Подписка на CVE активна
- [ ] Runbook инцидентов обновлён
```
