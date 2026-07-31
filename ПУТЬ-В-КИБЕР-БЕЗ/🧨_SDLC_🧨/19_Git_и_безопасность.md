# Git и безопасность

> AppSec-руководство · раздел 19/22 · сформировано 31.07.2026
> Полное руководство: см. остальные файлы в папке SDLC


### 19.1 Основные команды и различия

- `git pull` = `git fetch` + `git merge` – забрать изменения из remote и слить в текущую ветку
- `git merge` – объединение веток (создаёт merge commit, сохраняет историю обеих веток)
- `git rebase` – перекладывание коммитов текущей ветки поверх другой (линейная история, переписывает коммиты)
- `git cherry-pick` – перенос конкретных коммитов
- `git revert` – откат через новый коммит (безопасно для общей истории)
- `git reset` – откат локальной истории (опасно для общей)

**Merge vs Rebase (просто):**
- **merge:** история ветвистая, но сохраняется контекст "когда что сливали"
- **rebase:** история линейная, чистая, но переписывает коммиты (нельзя для public-веток)

### 19.2 Безопасность в Git

- **Не хранить секреты** (ключи, пароли, токены) в репозитории!
- **.gitignore:** `*.env`, `.env`, `*.pem`, `*.key`, `config/credentials*`, `secrets.*`, `*.pfx`
- **git-secrets / gitleaks / trufflehog** – блокировка коммитов с секретами (pre-commit hook, CI)
- Если секрет попал в историю – `git filter-repo` для очистки ИСТОРИИ (недостаточно удалить файл)
- **Подпись коммитов** (GPG/SSH) – verify signed commits
- **Branch protection** – запрет force-push на main, обязательные review
- **Signed tags**, проверка целостности зависимостей (lock-файлы)
> **GPG** – GNU Privacy Guard (программа шифрования и цифровой подписи)

> **SSH** – Secure Shell (защищённая оболочка (протокол удалённого доступа))


**Пример .gitignore для безопасности:**
```gitignore
.env
*.pem
*.key
*.p12
*.pfx
secrets.yaml
credentials.json
```


---

## 📎 Пример: .gitignore для безопасности (шаблон)

```gitignore
# Секреты и конфиги
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.pfx
*.jks
secrets.*
credentials.json
service-account*.json
*.secret

# IDE и ОС
.idea/
.vscode/
.DS_Store
*.swp

# Зависимости и сборка
node_modules/
vendor/
dist/
build/
__pycache__/
*.pyc

# Логи и БД (могут содержать данные)
*.log
*.sqlite3
*.db

# Инструменты безопасности
semgrep-results.*
gitleaks-report.*
trivy-*.json
```

---

## 📎 Пример: pre-commit hook для поиска секретов (gitleaks)

```bash
# .git/hooks/pre-commit (или через pre-commit framework)

#!/bin/sh
# Проверка на утечку секретов перед коммитом
if command -v gitleaks >/dev/null 2>&1; then
    gitleaks protect --staged --verbose
    if [ $? -ne 0 ]; then
        echo "❌ Найдены секреты! Коммит отменён."
        exit 1
    fi
else
    echo "⚠️ gitleaks не установлен, пропускаю проверку"
fi
```

```bash
# Установка gitleaks (macOS)
brew install gitleaks

# Проверка всей истории репозитория
gitleaks detect --source . --report-format json --report-path gitleaks-report.json

# Проверка staged изменений
gitleaks protect --staged
```

---

## 📎 Пример: если секрет уже попал в git-историю

```bash
# 1. НЕДОСТАТОЧНО просто удалить файл и закоммитить!
#    Секрет останется в истории.

# 2. Правильно: очистить историю через git filter-repo
pip install git-filter-repo

# Удалить файл из ВСЕЙ истории
git filter-repo --path .env --invert-paths

# Или заменить строку-секрет во всей истории
git filter-repo --replace-text <(echo "AKIAIOSFODNN7EXAMPLE==>REDACTED")

# 3. Срочно:
#    - отозвать/перевыпустить секрет (ротация ключа!)
#    - добавить секрет в .gitignore
#    - сообщить команде
# 4. Force push с осторожностью (переписывание истории)
git push --force-with-lease origin main
```

---

## 📎 Пример: merge vs rebase (наглядно)

```text
MERGE (история ветвистая, сохраняет контекст)
main:   A---B---C---D---E
                    \
feature:            F---G
                      \
main после merge:     D---M  (merge commit M объединяет F,G)

REBASE (история линейная, переписывает коммиты)
main:   A---B---C
feature:    F---G   (сделано от B)
rebase: F'---G'     (переложены на C)
main:   A---B---C---F'---G'  (линейно)
```

**Правила:**
- `rebase` – только для локальных/личных веток (переписывает историю)
- `merge` – для общих веток (main, develop)
- `pull` = `fetch` + `merge` (можно `pull --rebase` для чистоты)

**Защита main (branch protection):**
- Запрет force-push на main
- Обязательный MR + review (минимум 1-2 апрува)
- Обязательные CI-проверки (включая SAST/SCA)
- Подпись коммитов (GPG/SSH) – signed commits
