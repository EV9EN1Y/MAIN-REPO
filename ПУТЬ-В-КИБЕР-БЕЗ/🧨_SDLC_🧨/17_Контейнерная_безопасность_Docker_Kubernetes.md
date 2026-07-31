# Контейнерная безопасность: Docker, Kubernetes

> AppSec-руководство · раздел 17/22 · сформировано 31.07.2026
> Полное руководство: см. остальные файлы в папке SDLC


### 17.1 Основы Docker

**Dockerfile** – текстовый файл с инструкциями для сборки образа.

**Основные инструкции:**
```dockerfile
FROM ubuntu:22.04        # базовый образ
LABEL maintainer="me"    # метаданные
RUN apt-get update       # выполнение команд при сборке
COPY . /app              # копирование файлов
WORKDIR /app             # рабочая директория
ENV PORT=8080            # переменные окружения
EXPOSE 8080              # открытие порта
USER appuser             # пользователь (важно для безопасности!)
CMD ["python", "app.py"] # команда запуска
```

**Образ vs контейнер:**
- **Образ (Image)** – неизменяемый шаблон (read-only), из которого создаются контейнеры
- **Контейнер (Container)** – запущенный экземпляр образа (read-write слой поверх образа)

**Жизненный цикл:** Dockerfile → `docker build` → образ → `docker run` → контейнер

### 17.2 Уязвимости контейнеров

1. **Уязвимости в базовом образе** – старые пакеты с CVE → сканировать (Trivy, Grype)
2. **Уязвимости зависимостей приложения** – SCA
3. **Секреты в образе** – ENV/ARG/COPY с паролями, слои истории → многоступенчатая сборка (multi-stage)
4. **Запуск от root** – контейнер от root = при компрометации root на хосте → USER non-root
5. **Привилегированный режим (privileged)** – `--privileged` даёт доступ ко всем устройствам хоста, часто = полный контроль над хостом
6. **Открытые порты/сети** – лишние EXPOSE, host network
7. **Unpinned base images** – `FROM ubuntu:latest` = недетерминированная сборка → использовать точные теги/диджесты
8. **Небезопасные capabilities** – NET_ADMIN, SYS_ADMIN и др. → drop all + add needed
> **CVE** – Common Vulnerabilities and Exposures (общие уязвимости и факторы воздействия (реестр известных уязвимостей))

> **SCA** – Software Composition Analysis (анализ состава программного обеспечения)


### 17.3 Container Escape (побег из контейнера)

**Container Escape** – получение доступа за пределы контейнера (к хосту или другим контейнерам).

**Векторы:**
- Привилегированный режим + mount хостовой ФС
- CVE в runtime (runc CVE-2019-5736, CVE-2024-21626 – runc/leaky-vessels)
- Capabilities (SYS_ADMIN, CAP_SYS_PTRACE)
- Misconfigured Docker socket (/var/run/docker.sock внутри контейнера)
- Kernel exploits (не изолированы полностью – общий kernel)

**Защита:**
- Никогда не давать `--privileged`
- Ограничивать capabilities (`--cap-drop ALL --cap-add NET_BIND_SERVICE`)
- Не монтировать docker.sock
- Read-only rootfs (`--read-only`)
- Безопасные seccomp/AppArmor/SELinux профили
- Обновлять runtime (containerd, runc, Docker)
- **Pod Security Standards / Pod Security Admission** (K8s)
> **K8s** – Kubernetes (система оркестрации контейнеров)


### 17.4 Безопасность Kubernetes

- **RBAC** – минимальные права (least privilege)
- **NetworkPolicies** – сегментация сети
- **Pod Security** – ограничения на поды (privileged, hostPath, capabilities)
- **Secrets** – не в env/ConfigMap, использовать Secret + encryption at rest
- **Image scanning** – сканирование образов перед деплоем (Harbor/Trivy в CI)
- **Supply chain:** подпись образов (Cosign), SBOM, запрет latest-тегов
- **Runtime security:** Falco, Tetragon – обнаружение аномалий
> **RBAC** – Role-Based Access Control (управление доступом на основе ролей)

> **SBOM** – Software Bill of Materials (ведомость (спецификация) состава программного обеспечения)


---

## 📎 Пример: безопасный Dockerfile (шаблон)

```dockerfile
# Этап 1: сборка (builder)
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Этап 2: рантайм (безопасный)
FROM python:3.12-slim AS runtime

# не запускаться от root
RUN groupadd -r app && useradd -r -g app app

WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .

# минимальные права на файлы
RUN chown -R app:app /app && chmod -R 750 /app

# переменные окружения (не секреты в ENV!)
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

USER app

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
```

**Правила безопасности Dockerfile:**
1. `USER non-root` – обязательно (иначе root внутри = root на хосте при эскейпе)
2. Не хранить секреты в `ENV`/`ARG` – только через secrets/переменные окружения при запуске
3. Фиксировать версии базовых образов (`python:3.12-slim`, не `latest`)
4. Multi-stage сборка – не тащить компиляторы и исходники в прод
5. Минимизировать слои, ставить обновления (`apt-get update && upgrade`)
6. `--cap-drop ALL` при запуске, read-only FS

---

## 📎 Пример: отчёт Trivy по контейнеру (реальный формат)

```bash
$ trivy image --severity HIGH,CRITICAL myapp:latest

myapp:latest (debian 12.5)
==========================
Total: 7 (HIGH: 6, CRITICAL: 1)

┌───────────────┬──────────────────┬──────────┬──────────────────┬──────────┐
│    Library    │  Vulnerability   │ Severity │ Installed Version│ Fixed    │
├───────────────┼──────────────────┼──────────┼──────────────────┼──────────┤
│ libssl3       │ CVE-2026-XXXX1   │ CRITICAL │ 3.0.11-1         │ 3.0.13-1 │
│ libc6         │ CVE-2026-XXXX2   │ HIGH     │ 2.36-9           │ 2.36-10  │
│ openssl       │ CVE-2026-XXXX3   │ HIGH     │ 3.0.11-1         │ 3.0.13-1 │
│ curl          │ CVE-2026-XXXX4   │ HIGH     │ 8.4.0            │ 8.5.0    │
└───────────────┴──────────────────┴──────────┴──────────────────┴──────────┘

Application dependencies (pip)
=============================
┌──────────┬──────────────────┬──────────┬──────────────────┬──────────┐
│ fastapi  │ CVE-2026-XXXX5   │ HIGH     │ 0.110.0          │ 0.112.0  │
└──────────┴──────────────────┴──────────┴──────────────────┴──────────┘
```

**Что делать:**
- Critical в базовом образе: обновить base image (`apt-get upgrade` в Dockerfile)
- Critical в зависимостях: обновить пакет (см. Fixed version)
- Настроить блокировку: `trivy image --exit-code 1 --severity CRITICAL`

---

## 📎 Пример: безопасный запуск контейнера (команды)

```bash
# Безопасный запуск: без привилегий, без лишних capabilities
docker run -d \
  --name app \
  --read-only \                       # read-only rootfs
  --cap-drop ALL \                    # убрать ВСЕ capabilities
  --cap-add NET_BIND_SERVICE \        # вернуть только нужные
  --security-opt no-new-privileges \  # запрет повышения привилегий
  --pids-limit 100 \                  # лимит процессов
  --memory 512m \                     # лимит памяти
  -e SECRET_KEY_FILE=/run/secrets/key \
  --secret key,source=/etc/secrets/key \
  registry.example.com/app:1.0.0

# НИКОГДА так:
# docker run --privileged ...        <- полный доступ к хосту
# docker run -v /:/host ...          <- монтирование всей ФС хоста
# docker run -v /var/run/docker.sock:/var/run/docker.sock ...  <- контроль над Docker
```

---

## 📎 Пример: политика Pod Security (Kubernetes)

```yaml
# Pod Security Standards: restricted (пример)
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true            # не root
    seccompProfile:
      type: RuntimeDefault        # seccomp
  containers:
    - name: app
      image: registry.example.com/app:1.0.0
      securityContext:
        allowPrivilegeEscalation: false   # запрет эскалации
        capabilities:
          drop: ["ALL"]                   # убрать все capabilities
        readOnlyRootFilesystem: true      # read-only ФС
        runAsUser: 1000
      resources:
        limits:
          memory: "512Mi"
          cpu: "500m"
```
