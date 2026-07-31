# OWASP Top 10

> AppSec-руководство · раздел 16/22 · сформировано 31.07.2026
> Полное руководство: см. остальные файлы в папке SDLC


**OWASP Top 10** – стандартный документ-«осознание рисков» для разработчиков и специалистов по безопасности веб-приложений. Обновляется раз в несколько лет (2017, 2021, 2025).
> **OWASP** – Open Web Application Security Project (открытый проект безопасности веб-приложений)


### OWASP Top 10 – 2021

1. **A01:2021 – Broken Access Control (нарушенный контроль доступа)**
   - IDOR, missing authorization checks, path traversal
   - Контрмеры: авторизация на каждом запросе, deny by default, проверка прав
> **IDOR** – Insecure Direct Object Reference (небезопасная прямая ссылка на объект)


2. **A02:2021 – Cryptographic Failures (криптографические сбои)**
   - Слабые алгоритмы, отсутствие шифрования, хардкод ключей
   - Контрмеры: сильные алгоритмы (AES-256, TLS 1.2+), правильное управление ключами
> **AES** – Advanced Encryption Standard (стандарт симметричного шифрования)

> **TLS** – Transport Layer Security (протокол безопасности транспортного уровня)


3. **A03:2021 – Injection (инъекции)**
   - SQLi, NoSQLi, OS Command Injection, XSS (в 2021 перенесён в инъекции)
   - Контрмеры: prepared statements, параметризация, валидация ввода, escaping
> **OS** – Operating System (операционная система)

> **XSS** – Cross-Site Scripting (межсайтовый скриптинг)

> **SQLi** – SQL Injection (инъекция SQL)


4. **A04:2021 – Insecure Design (небезопасный дизайн)**
   - Архитектурные проблемы, отсутствие threat modeling
   - Контрмеры: threat modeling, secure design patterns, security requirements

5. **A05:2021 – Security Misconfiguration (ошибки конфигурации)**
   - Дефолтные учётки, открытые порты, verbose errors, лишние функции
   - Контрмеры: hardened конфигурации, автоматизация (IaC), регулярные аудиты
> **IaC** – Infrastructure as Code (инфраструктура как код)


6. **A06:2021 – Vulnerable and Outdated Components (уязвимые компоненты)**
   - Устаревшие библиотеки с CVE
   - Контрмеры: SCA, SBOM, патчинг, Dependency-Track
> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))

> **SBOM** – Software Bill of Materials (ведомость (спецификация) состава программного обеспечения)

> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)


7. **A07:2021 – Identification and Authentication Failures (сбои идентификации/аутентификации)**
   - Слабые пароли, отсутствие MFA, session fixation
   - Контрмеры: MFA, password policies, secure session management
> **MFA** – Multi-Factor Authentication (многофакторная аутентификация)


8. **A08:2021 – Software and Data Integrity Failures (нарушение целостности)**
   - Небезопасная десериализация, недоверенные обновления, подпись кода
   - Контрмеры: проверка подписей, безопасная десериализация, SBOM

9. **A09:2021 – Security Logging and Monitoring Failures (отсутствие логирования/мониторинга)**
   - Нет логов, нет алертов, инциденты не обнаруживаются
   - Контрмеры: централизованное логирование, SIEM, алерты, мониторинг
> **SIEM** – Security Information and Event Management (управление событиями и информацией о безопасности)


10. **A10:2021 – SSRF (Server-Side Request Forgery)**
    - Сервер выполняет запросы по контролируемому URL (внутренние сервисы)
    - Контрмеры: валидация URL, запрет внутренних адресов, сетевые сегментации
> **URL** – Uniform Resource Locator (унифицированный указатель ресурса)

> **SSRF** – Server-Side Request Forgery (подделка серверных запросов)


### OWASP Top 10 – 2025 (новинки)

Актуальная версия 2025 года. Ключевые изменения по сравнению с 2021:
- Появились новые категории: **AI/LLM-related risks**, **Insecure Industry-Specific Applications**
- Усилен фокус на: **Improper Access Control** (A01), **Misconfiguration**, **Vulnerable Components**
- Отдельно выделены риски, связанные с LLM/AI (prompt injection и др.)

*Примечание: точный список Top 10 2025 уточняйте на owasp.org – документ регулярно обновляется.*


---

## 📎 Пример: описание уязвимости по OWASP Top 10 (шаблон)

> Как оформлять найденную уязвимость, чтобы разработчик понял и исправил.
> Пример для A01:2021 Broken Access Control (IDOR).

```markdown
# Уязвимость: IDOR в /api/v1/posts/{id}

## Классификация
- OWASP Top 10 2021: A01:2021 - Broken Access Control
- CWE: CWE-639 (Authorization Bypass Through User-Controlled Key)
- Severity: High (CVSS 8.1: AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N)

## Описание
Эндпоинт /api/v1/posts/{id} возвращает пост по id БЕЗ проверки,
принадлежит ли пост текущему пользователю. Любой авторизованный
пользователь может читать/менять/удалять чужие посты, подставляя id.

## Как воспроизвести (PoC)
1. Залогиниться как user_A
2. GET /api/v1/posts/12345  (id поста user_B)
3. Ответ: 200 OK + данные чужого поста  <- уязвимость

## Пример запроса
```http
GET /api/v1/posts/12345 HTTP/1.1
Host: api.example.com
Authorization: Bearer <token_user_A>
```

## Пример ответа (уязвимого)
```json
{
  "id": 12345,
  "author_id": 999,
  "text": "личный пост другого пользователя",
  "is_private": true
}
```

## Причина
- Нет проверки author_id == текущий user_id
- Доступ к объекту по контролируемому id (IDOR)

## Исправление (рекомендация)
```python
# Было (уязвимо):
@app.get("/api/v1/posts/{post_id}")
def get_post(post_id: int):
    post = db.query(Post).filter(Post.id == post_id).first()
    return post

# Надо (безопасно):
@app.get("/api/v1/posts/{post_id}")
def get_post(post_id: int, user=Depends(get_current_user)):
    post = db.query(Post).filter(Post.id == post_id).first()
    if not post:
        raise HTTPException(404)
    # проверка прав: владелец или публичный пост
    if post.author_id != user.id and post.is_private:
        raise HTTPException(403, "Нет доступа")
    return post
```

## Проверка исправления
1. Повторить PoC: ожидаем 403/404
2. Прогнать тест: пользователь не видит чужие приватные посты
3. SAST-правило: проверка author_id в каждом запросе к объекту
```

---

## 📎 Пример: чек-лист проверки по OWASP Top 10 (мини)

| # | Категория | Как проверить быстро |
|---|-----------|----------------------|
| A01 | Access Control | Подменить id/роль в запросе -> ожидать 403 |
| A02 | Crypto | Проверить TLS, алгоритмы хэшей, ключи не в коде |
| A03 | Injection | Отправить ' OR 1=1, <script>, $(), |id в поля |
| A04 | Insecure Design | Есть ли threat modeling, security requirements |
| A05 | Misconfiguration | DEBUG, дефолтные пароли, лишние порты |
| A06 | Vulnerable Components | SCA (Trivy/Dependency-Check) |
| A07 | Auth | Брутфорс, слабые пароли, MFA, session fixation |
| A08 | Integrity | Подпись обновлений, безопасная десериализация |
| A09 | Logging | Логируются входы/админ-действия? Есть алерты? |
| A10 | SSRF | URL-параметры: подставить http://169.254.169.254 |
