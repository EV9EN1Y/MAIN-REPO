# OWASP MASVS v2.1.0 – Чек-лист 

**Mobile Application Security Verification Standard** – стандарт требований безопасности мобильных приложений
Источник: `OWASP_MASVS.pdf` (v2.1.0, январь 2024) | Дата составления: 04.08.2026

> 8 категорий (controls), 23 требования. Термины: английские, пояснения – на русском.
> Для каждого пункта тест-кейсы («как проверить») находятся в **OWASP MASTG** (Mobile Application Security Testing Guide).

---

## Как пользоваться чек-листом

1. Для каждого требования проставить статус:
   - **✅ Pass** – выполнено (проверено по MASTG)
   - **❌ Fail** – не выполнено (есть замечание)
   - **➖ N/A** – неприменимо (функция отсутствует в приложении)
2. В колонке «Комментарий» пиши доказательство/инструмент (MobSF, Frida, Burp, код-ревью).
3. Профили тестирования MASVS: **Standard** (базовый), **Extended** (чувствительные данные), **Resilience** (защита от реверса). Чем выше профиль – тем больше требований применяется.

---

## 1. MASVS-STORAGE – Хранение данных (Storage)

> О чём: приложение хранит много чувствительных данных (PII, криптоматериалы, секреты, API-ключи) локально. Категория про защиту намеренно хранимых данных и про непреднамеренные утечки (через API, бэкапы, логи).

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-STORAGE-1 | Приложение **безопасно хранит чувствительные данные** (sensitive data) – независимо от места: приватное хранилище (internal storage) или публичные папки | The app securely stores sensitive data | ☐ | |
| MASVS-STORAGE-2 | Приложение **предотвращает утечку чувствительных данных** (data leakage) – непреднамеренное сохранение/раскрытие через API, бэкапы (backups), логи (logs) | The app prevents leakage of sensitive data | ☐ | |

## 2. MASVS-CRYPTO – Криптография (Cryptography)

> О чём: устройства легко теряются/крадутся, поэтому данные шифруются. Категория про использование криптографии по индустриальным стандартам (NIST SP 800-175B, NIST SP 800-57) и про управление ключами.

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-CRYPTO-1 | Приложение использует **современную сильную криптографию** (strong cryptography) и применяет её по лучшим практикам индустрии | The app employs current strong cryptography and uses it according to industry best practices | ☐ | |
| MASVS-CRYPTO-2 | Приложение выполняет **управление ключами** (key management) по лучшим практикам: генерация, хранение, защита на всём жизненном цикле | The app performs key management according to industry best practices | ☐ | |

## 3. MASVS-AUTH – Аутентификация и авторизация (Authentication and Authorization)

> О чём: аутентификация (биометрия, PIN, MFA) и авторизация. Важно: enforcement (принудительная проверка) должен быть на сервере (remote endpoint), а клиент обязан безопасно использовать протоколы. Серверная часть проверяется по OWASP ASVS.

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-AUTH-1 | Приложение использует **безопасные протоколы аутентификации и авторизации** (authentication/authorization protocols) и следует лучшим практикам | The app uses secure authentication and authorization protocols and follows the relevant best practices | ☐ | |
| MASVS-AUTH-2 | Приложение выполняет **локальную аутентификацию** (local authentication: биометрия, PIN) безопасно, по платформенным практикам | The app performs local authentication securely according to the platform best practices | ☐ | |
| MASVS-AUTH-3 | Приложение защищает **чувствительные операции дополнительной аутентификацией** (additional authentication: biometric, PIN, MFA-код, email, deep links) | The app secures sensitive operations with additional authentication | ☐ | |

## 4. MASVS-NETWORK – Сетевое взаимодействие (Network Communication)

> О чём: конфиденциальность и целостность данных в пути (data in transit). Обычно это TLS + аутентификация удалённого эндпоинта. Категория про то, что нельзя отключать безопасные настройки платформы по умолчанию и что можно сузить доверие только к своим CA (pinning).

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-NETWORK-1 | Приложение **защищает весь сетевой трафик** (network traffic) по актуальным лучшим практикам (TLS, шифрование канала) | The app secures all network traffic according to the current best practices | ☐ | |
| MASVS-NETWORK-2 | Приложение выполняет **pinning идентичности** (identity pinning: certificate pinning / public key pinning) для всех удалённых эндпоинтов под контролем разработчика | The app performs identity pinning for all remote endpoints under the developer's control | ☐ | |

## 5. MASVS-PLATFORM – Взаимодействие с платформой (Platform Interaction)

> О чём: приложение взаимодействует с платформой через IPC и WebView, показывает чувствительные данные в UI. Категория про безопасное использование этих механизмов и про защиту от утечек через скриншоты, уведомления, shoulder surfing.

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-PLATFORM-1 | Приложение **безопасно использует механизмы IPC** (inter-process communication: интенты, экранирование) | The app uses IPC mechanisms securely | ☐ | |
| MASVS-PLATFORM-2 | Приложение **безопасно использует WebViews** – конфигурация без утечек и без опасных JavaScript-мостов (JS bridges) к нативному коду | The app uses WebViews securely | ☐ | |
| MASVS-PLATFORM-3 | Приложение **безопасно использует пользовательский интерфейс** (UI): пароли, данные карт, OTP в уведомлениях не должны утекать через автогенерируемые скриншоты, shoulder surfing, передачу устройства | The app uses the user interface securely | ☐ | |

## 6. MASVS-CODE – Качество кода (Code Quality)

> О чём: точки входа данных (UI, IPC, сеть, файловая система) должны обрабатывать ввод как недоверенный. Плюс актуальность платформы, принудительные обновления и отсутствие известных уязвимостей в компонентах. Рекомендуемые практики: OWASP SAMM, NIST SP 800-218 (SSDF).

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-CODE-1 | Приложение **требует актуальную версию платформы** (up-to-date platform version) – не поддерживает заведомо уязвимые старые ОС | The app requires an up-to-date platform version | ☐ | |
| MASVS-CODE-2 | Приложение имеет **механизм принудительного обновления** (enforcing app updates) – возможность заставить пользователя обновиться при критичной уязвимости | The app has a mechanism for enforcing app updates | ☐ | |
| MASVS-CODE-3 | Приложение использует **только компоненты без известных уязвимостей** (known vulnerabilities) – сканирование зависимостей (SCA), библиотеки | The app only uses software components without known vulnerabilities | ☐ | |
| MASVS-CODE-4 | Приложение **валидирует и санитизирует все недоверенные входные данные** (untrusted inputs) – защита от SQL injection, XSS, insecure deserialization, обхода проверок | The app validates and sanitizes all untrusted inputs | ☐ | |

## 7. MASVS-RESILIENCE – Устойчивость к реверсу и модификации (Resilience Against Reverse Engineering and Tampering)

> О чём: defense-in-depth против реверса: обфускация (obfuscation), анти-отладка (anti-debugging), анти-тамперинг (anti-tampering). Важно: отсутствие этих мер не создаёт уязвимость само по себе – это дополнительная защита под конкретную угрозу (кража IP, читы, бэкдоры).

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-RESILIENCE-1 | Приложение **проверяет целостность платформы** (integrity of the platform) – детект root/jailbreak, компрометированной ОС | The app validates the integrity of the platform | ☐ | |
| MASVS-RESILIENCE-2 | Приложение реализует **механизмы анти-тамперинга** (anti-tampering) – защита целостности кода и ресурсов, детект модифицированных версий | The app implements anti-tampering mechanisms | ☐ | |
| MASVS-RESILIENCE-3 | Приложение реализует **механизмы против статического анализа** (anti-static analysis) – обфускация кода, запутывание логики | The app implements anti-static analysis mechanisms | ☐ | |
| MASVS-RESILIENCE-4 | Приложение реализует **техники против динамического анализа** (anti-dynamic analysis) – анти-отладка, защита от instrumentation (Frida и аналоги) | The app implements anti-dynamic analysis techniques | ☐ | |

## 8. MASVS-PRIVACY – Приватность (Privacy)

> О чём: базовый уровень приватности с точки зрения самого приложения (не заменяет DPIA по GDPR/ENISA). Про минимизацию данных, согласие, прозрачность, контроль пользователя и защиту от идентификации (fingerprinting).

| ID | Требование (RU) | Оригинал (EN) | Статус | Комментарий |
|---|---|---|---|---|
| MASVS-PRIVACY-1 | Приложение **минимизирует доступ к чувствительным данным и ресурсам** (data minimization) – только необходимые данные, согласие (consent), контроль third-party SDK, учёт цепочки поставок (supply chain, SBOM) | The app minimizes access to sensitive data and resources | ☐ | |
| MASVS-PRIVACY-2 | Приложение **предотвращает идентификацию пользователя** (user identification) – анонимизация, псевдонимизация, изоляция fingerprint-сигналов (device ID, IP, поведенческие паттерны) | The app prevents identification of the user | ☐ | |
| MASVS-PRIVACY-3 | Приложение **прозрачно в сборе и использовании данных** (transparency) – понятная информация о сборе/хранении/передаче, включая фоновый сбор (background data collection) | The app is transparent about data collection and usage | ☐ | |
| MASVS-PRIVACY-4 | Приложение **даёт пользователю контроль над данными** (user control) – управление, удаление, изменение данных, отзыв согласия (revoke consent), повторный запрос согласия при расширении сбора | The app offers user control over their data | ☐ | |

---

## Сводка по проверке

| Категория | Всего | ✅ Pass | ❌ Fail | ➖ N/A | Заметок |
|---|---|---|---|---|---|
| MASVS-STORAGE | 2 | | | | |
| MASVS-CRYPTO | 2 | | | | |
| MASVS-AUTH | 3 | | | | |
| MASVS-NETWORK | 2 | | | | |
| MASVS-PLATFORM | 3 | | | | |
| MASVS-CODE | 4 | | | | |
| MASVS-RESILIENCE | 4 | | | | |
| MASVS-PRIVACY | 4 | | | | |
| **Итого** | **23** | | | | |

## Полезные ссылки

- MASVS онлайн: https://mas.owasp.org/MASVS
- MASTG (тест-кейсы к каждому пункту): https://mas.owasp.org/MASTG
- MAS Checklist (Excel): https://mas.owasp.org/checklist
- Инструменты: MobSF, Frida, objection, Burp Suite

---

*Чек-лист составлен по OWASP MASVS v2.1.0 (PDF). Заполняется при аудите мобильного приложения; тест-кейсы – в MASTG.*
