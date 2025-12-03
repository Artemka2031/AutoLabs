# GitLab CI/CD для AutoLabs

## Дата создания
2025-12-01

---

## О компоненте

### Что это?
**GitLab** - платформа для управления кодом, CI/CD и Container Registry. Центральный hub для разработки и деплоя AutoLabs.

### Где разворачивается?
- **Тип:** Виртуальная машина (VM) в Proxmox
- **Размещение:** Отдельно от Kubernetes кластера
- **Доступ:** https://registry.gitlab.local

### Ресурсы

| Параметр | Значение |
|----------|----------|
| **CPU** | 4 cores |
| **RAM** | 8 GB |
| **Диск** | 100 GB SSD |
| **ОС** | Ubuntu 22.04 LTS |
| **Сеть** | Статичный IP, доступ из K8s и внешней сети |

### Компоненты на VM

| Компонент | Назначение | Порт |
|-----------|-----------|------|
| **GitLab CE/EE** | Git repositories, Web UI, Projects | :443 (HTTPS), :22 (SSH) |
| **GitLab Container Registry** | Docker образов хранилище | :5050 |
| **GitLab Runner** | CI/CD executor (Docker) | - |
| **PostgreSQL** | Database для GitLab | :5432 (localhost) |

### Зависимости
- **Kubernetes Cluster** - деплой приложений через GitLab CI
- **VM Database** - опционально (GitLab может использовать свою локальную PostgreSQL)
- **Telegram Bot** - уведомления о pipelines

---

## Бизнес-логика и назначение

### Зачем нужен GitLab?

**Проблема без GitLab:**
- Нет централизованного хранилища кода (разработчики работают изолированно)
- Ручной билд Docker образов (медленно, подвержено ошибкам)
- Ручной деплой в K8s (риск человеческого фактора)
- Нет версионирования образов (сложно откатиться к предыдущей версии)
- Нет автоматизации проверки безопасности (уязвимости могут попасть в production)

**Решение - GitLab:**
1. **Version Control** - Git репозитории для всего кода платформы
2. **Автоматизация билда** - от commit до Docker образа автоматически
3. **Security scanning** - Trivy проверяет каждый образ на уязвимости
4. **Автоматический деплой** - Helm upgrade в K8s без ручных действий
5. **Container Registry** - единое хранилище всех образов (platform + labs)
6. **Прозрачность** - история изменений, кто что когда изменил

### Вклад в работу платформы

**GitLab обеспечивает:**

**Для разработчиков платформы:**
- ✅ Git workflow (branches, merge requests, code review)
- ✅ Автоматический билд при каждом commit
- ✅ Автоматический деплой в K8s
- ✅ Уведомления о статусе pipeline (Telegram)

**Для преподавателей (косвенно):**
- ✅ Хранилище образов лабораторных работ
- ✅ Автоматическая сборка лаб после одобрения Admin
- ✅ Версионирование лаб (v1, v2, latest)

**Для Admin:**
- ✅ Контроль качества (security scan перед публикацией)
- ✅ Управление доступом (кто может пушить в какие группы)
- ✅ Аудит изменений (кто загрузил новую лабу)

**Для платформы в целом:**
- ✅ Единый источник истины (все образы в одном месте)
- ✅ Reproducibility (можно пересобрать любую версию)
- ✅ Disaster recovery (backup репозиториев)

---

## Обоснование выбора GitLab

### Почему GitLab, а не альтернативы?

**Альтернативы:**
- **GitHub + GitHub Actions** - облачный, зависимость от внешнего сервиса, платные runners
- **Jenkins + Gitea/Gogs** - отдельные системы (Git + CI/CD), сложнее интеграция
- **GitLab Cloud** - платный, нет контроля над данными
- **Drone CI + Gitea** - меньше features "из коробки"

**Преимущества GitLab:**
1. **Всё-в-одном** - Git + CI/CD + Container Registry + Issue Tracking
2. **Self-hosted** - полный контроль, приватность данных
3. **Open-source** - GitLab CE бесплатный
4. **Простота** - одна установка, единый интерфейс
5. **Богатый CI/CD** - мощный .gitlab-ci.yml с stages, artifacts, environments
6. **Встроенный Registry** - не нужен отдельный Harbor/Nexus
7. **Kubernetes интеграция** - native поддержка деплоя в K8s

**Требования AutoLabs → GitLab:**

| Требование | GitLab | GitHub Actions | Jenkins + Gitea |
|------------|--------|---------------|----------------|
| Self-hosted | ✅ | ⚠️ (платно) | ✅ |
| Container Registry | ✅ | ❌ (отдельно) | ❌ (отдельно) |
| Интеграция Git + CI/CD | ✅ Нативная | ✅ | ⚠️ Ручная |
| Helm deploy | ✅ | ✅ | ⚠️ Плагины |
| Простота setup | ✅ | N/A | ❌ (две системы) |
| Open-source | ✅ CE | ❌ | ✅ |
| UI/UX | ✅ Отличный | ✅ | ⚠️ Устаревший |

**Вывод:** GitLab - оптимальное решение для self-hosted платформы с полным циклом от кода до production.

---

## Обоснование размещения вне K8s

### Почему GitLab на отдельной VM, а не в K8s?

**Альтернатива:** GitLab внутри K8s (через Helm chart)

**Проблемы с GitLab в K8s:**
1. **Высокое потребление ресурсов:**
   - GitLab требует минимум 4 CPU + 8 GB RAM
   - Это 1/3 всего K8s кластера (12 CPU, 16 GB)
   - Остаётся мало ресурсов для лабораторных работ

2. **Dependency loop (циклическая зависимость):**
   - GitLab CI деплоит приложения В K8s
   - Но сам GitLab ВНУТРИ K8s
   - Проблема при падении кластера: нельзя задеплоить GitLab через GitLab

3. **Сложность восстановления:**
   - K8s кластер упал → GitLab недоступен
   - Невозможно задеплоить исправления через CI/CD
   - Нужно восстанавливать вручную

4. **Производительность:**
   - GitLab на VM работает быстрее (нет оверхеда K8s)
   - Прямой доступ к диску (не через PersistentVolume)

**Преимущества GitLab на отдельной VM:**
- ✅ Независимость от K8s (K8s упал → GitLab работает)
- ✅ Не тратит ресурсы кластера (все 12 CPU для приложений)
- ✅ Проще backup (snapshot VM в Proxmox)
- ✅ Проще восстановление (restore VM snapshot)
- ✅ Производительность (bare-metal PostgreSQL, диск)

**Вывод:** Отдельная VM для GitLab - архитектурно правильное решение для нашего масштаба (ограниченные ресурсы, один кластер).

---

## Альтернативный вариант: GitLab на VPS (аренда)

**Для экономии ресурсов локального сервера:**

| Параметр | Локальная VM | VPS (Hetzner) |
|----------|--------------|---------------|
| **Стоимость** | Бесплатно (свой сервер) | €10-15/мес |
| **Ресурсы локально** | Забирает 4 CPU, 8 GB | Освобождает для K8s |
| **Контроль данных** | ✅ Полный | ⚠️ У провайдера |
| **Доступность** | ⚠️ Зависит от локальной сети | ✅ Высокая (99.9%) |
| **Скорость** | ✅ Локальная сеть (быстро) | ⚠️ Интернет (медленнее) |

**Рекомендация для MVP:**
- **Если ресурсы 12 CPU, 16 GB:** GitLab на арендованном VPS (экономим для K8s)
- **Если ресурсы 24 CPU, 32 GB:** GitLab на локальной VM (достаточно для всех)

---

## Обзор

**GitLab** является центральным компонентом для:
- **Хранения кода** платформы и конфигураций лабораторных работ
- **Container Registry** - хранение Docker образов
- **CI/CD** - автоматическая сборка, тестирование и деплой
- **Автоматизации** процесса одобрения и публикации лабораторных работ

### Ключевые принципы:

1. **Разделение по группам** - platform (платформенные образы) и labs (лабораторные работы)
2. **Автоматизация** - от commit до production через GitLab CI
3. **Security-first** - Trivy сканирование всех образов
4. **Централизованные Helm charts** - единый репозиторий для деплоя
5. **Уведомления** - Telegram для критичных событий

---

## Структура GitLab

### Группы и проекты

```
GitLab Instance (registry.gitlab.local)
│
├── Group: autolabs/platform/
│   │
│   ├── Project: frontend
│   │   ├── src/ (React/Vue код)
│   │   ├── Dockerfile
│   │   ├── .gitlab-ci.yml
│   │   └── README.md
│   │
│   ├── Project: backend-platform
│   │   ├── src/ (FastAPI код)
│   │   ├── requirements.txt
│   │   ├── Dockerfile
│   │   ├── .gitlab-ci.yml
│   │   └── README.md
│   │
│   └── Project: celery-workers
│       ├── src/ (Celery tasks, K8s operations)
│       ├── requirements.txt
│       ├── Dockerfile
│       ├── .gitlab-ci.yml
│       └── README.md
│
├── Group: autolabs/labs/
│   │
│   ├── Project: sql-injection
│   │   ├── lab-config.yaml (описание лабы)
│   │   ├── Dockerfile
│   │   ├── app/ (уязвимое приложение)
│   │   ├── .gitlab-ci.yml
│   │   └── README.md
│   │
│   ├── Project: network-pentest
│   ├── Project: web-vulnerabilities
│   └── Project: malware-analysis
│
└── Group: autolabs/infrastructure/
    │
    └── Project: helm-charts
        ├── backend-platform/
        │   ├── Chart.yaml
        │   ├── values.yaml
        │   ├── values-dev.yaml
        │   ├── values-prod.yaml
        │   └── templates/
        │       ├── deployment.yaml
        │       ├── service.yaml
        │       └── ingress.yaml
        │
        ├── celery-workers/
        ├── frontend/
        ├── zitadel/
        └── rabbitmq/
```

---

## Права доступа к группам

### Group: autolabs/platform/

| Роль | GitLab Permission | Может |
|------|------------------|-------|
| **Разработчики** | Developer/Maintainer | Push код, Merge MR, Trigger pipelines |
| **Admin** | Owner | Всё + управление настройками |
| **CI/CD Service Account** | Developer | Автоматический деплой |

### Group: autolabs/labs/

| Роль | GitLab Permission | Может |
|------|------------------|-------|
| **Admin** | Owner | Всё + загрузка новых лаб |
| **Backend Platform Service Account** | Developer | Автоматические коммиты при одобрении заявок |
| **Teacher** | Reporter | Только чтение готовых лаб (см. [06-authorization](../06-authorization/README.md)) |

**Примечание:** Детальная логика доступа преподавателей к лабам описана в [06-authorization](../06-authorization/README.md#процесс-загрузки-лабораторных-работ).

### Group: autolabs/infrastructure/

| Роль | GitLab Permission | Может |
|------|------------------|-------|
| **DevOps/Admin** | Maintainer | Обновление Helm charts |
| **CI/CD Service Account** | Developer | Клонирование charts для деплоя |

---

## CI/CD Pipeline для platform образов

### Общая структура pipeline

```
Git Push → GitLab CI
    ↓
┌───────────────────────────────────┐
│ Stage 1: Build                    │
│ - Билд Docker образа              │
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│ Stage 2: Scan                     │
│ - Trivy security scan             │
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│ Stage 3: Push                     │
│ - Push в GitLab Container Registry│
└───────────────┬───────────────────┘
                ↓
┌───────────────────────────────────┐
│ Stage 4: Deploy                   │
│ - Clone helm-charts               │
│ - helm upgrade в K8s              │
└───────────────────────────────────┘
```

### Пример: .gitlab-ci.yml для backend-platform

```yaml
# autolabs/platform/backend-platform/.gitlab-ci.yml

stages:
  - build
  - scan
  - push
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY/autolabs/platform/backend-platform
  IMAGE_TAG: latest
  HELM_CHARTS_REPO: https://gitlab-ci-token:${CI_JOB_TOKEN}@registry.gitlab.local/autolabs/infrastructure/helm-charts.git

# Stage 1: Build Docker Image
build:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
    - docker save ${IMAGE_NAME}:${IMAGE_TAG} -o backend-platform.tar
  artifacts:
    paths:
      - backend-platform.tar
    expire_in: 1 hour
  only:
    - main
    - develop

# Stage 2: Security Scan with Trivy
scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --input backend-platform.tar --severity HIGH,CRITICAL --exit-code 0
    # exit-code 0: не падаем при найденных уязвимостях (только логируем)
    # для production можно --exit-code 1 (падать при CRITICAL)
  dependencies:
    - build
  allow_failure: false
  only:
    - main
    - develop

# Stage 3: Push to Registry
push:
  stage: push
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker load -i backend-platform.tar
    - docker push ${IMAGE_NAME}:${IMAGE_TAG}
  dependencies:
    - build
  only:
    - main

# Stage 4: Deploy to Kubernetes
deploy:
  stage: deploy
  image: alpine/helm:latest
  before_script:
    - apk add --no-cache git
    - git clone ${HELM_CHARTS_REPO} /tmp/helm-charts
  script:
    # Настройка kubectl для доступа к K8s
    - mkdir -p ~/.kube
    - echo "$KUBECONFIG_CONTENT" | base64 -d > ~/.kube/config

    # Деплой через Helm
    - |
      helm upgrade --install backend-platform /tmp/helm-charts/backend-platform \
        --namespace platform \
        --set image.tag=${IMAGE_TAG} \
        --set image.repository=${IMAGE_NAME} \
        -f /tmp/helm-charts/backend-platform/values.yaml \
        --wait \
        --timeout 5m
  only:
    - main
  when: manual  # Ручной запуск для безопасности

# Уведомления в Telegram (при failure)
notify_failure:
  stage: .post
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d "chat_id=${TELEGRAM_CHAT_ID}" \
        -d "text=❌ Pipeline FAILED: ${CI_PROJECT_NAME} (${CI_COMMIT_REF_NAME})\nCommit: ${CI_COMMIT_SHORT_SHA}\nAuthor: ${CI_COMMIT_AUTHOR}\nURL: ${CI_PIPELINE_URL}"
  when: on_failure
  only:
    - main

# Уведомления в Telegram (при успехе deploy)
notify_success:
  stage: .post
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d "chat_id=${TELEGRAM_CHAT_ID}" \
        -d "text=✅ Deploy SUCCESS: ${CI_PROJECT_NAME} → K8s\nVersion: ${IMAGE_TAG}\nCommit: ${CI_COMMIT_SHORT_SHA}\nAuthor: ${CI_COMMIT_AUTHOR}"
  when: on_success
  only:
    - main
  needs:
    - deploy
```

### Переменные CI/CD (GitLab Settings → CI/CD → Variables)

| Переменная | Описание | Тип |
|-----------|----------|-----|
| `KUBECONFIG_CONTENT` | Kubeconfig для доступа к K8s (base64) | File, Protected |
| `TELEGRAM_BOT_TOKEN` | Токен Telegram бота для уведомлений | Variable, Masked |
| `TELEGRAM_CHAT_ID` | ID чата для уведомлений | Variable |
| `CI_REGISTRY_USER` | Логин GitLab Registry | Built-in |
| `CI_REGISTRY_PASSWORD` | Пароль GitLab Registry | Built-in |
| `CI_REGISTRY` | URL GitLab Registry | Built-in |

---

## CI/CD Pipeline для celery-workers

### Отличия от backend-platform

Celery Workers - отдельный репозиторий с **инфраструктурной логикой**:
- Задачи для работы с K8s API
- Архивация volumes в Minio
- Обработка очередей RabbitMQ

**Pipeline идентичен backend-platform**, но:
- Другой `IMAGE_NAME`: `$CI_REGISTRY/autolabs/platform/celery-workers`
- Другой Helm chart: `helm-charts/celery-workers`
- Дополнительные environment variables для K8s доступа

```yaml
# Специфичные переменные для Celery Workers
deploy:
  script:
    - |
      helm upgrade --install celery-workers /tmp/helm-charts/celery-workers \
        --namespace infrastructure \
        --set image.tag=${IMAGE_TAG} \
        --set k8s.serviceAccount=celery-k8s-operator \
        --set rabbitmq.host=rabbitmq.messaging.svc.cluster.local \
        -f /tmp/helm-charts/celery-workers/values.yaml
```

---

## CI/CD Pipeline для frontend

### Особенности

Frontend (React/Vue) билдится в **Nginx контейнер со статикой**.

```yaml
# autolabs/platform/frontend/.gitlab-ci.yml

build:
  stage: build
  image: node:18-alpine
  script:
    # Билд статики
    - npm ci
    - npm run build

    # Билд Docker образа с Nginx
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
    - docker save ${IMAGE_NAME}:${IMAGE_TAG} -o frontend.tar
```

**Dockerfile для frontend:**

```dockerfile
# autolabs/platform/frontend/Dockerfile

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Nginx
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## CI/CD Pipeline для labs образов

### Процесс загрузки лабораторной работы

#### Вариант A: Admin загружает вручную (MVP)

```
1. Teacher создаёт заявку через Frontend:
   - Загружает lab-config.yaml
   - Загружает Dockerfile(s)
   - Загружает код приложения
   - Описание лабораторной работы

2. Backend Platform сохраняет заявку (статус: pending)

3. Admin получает уведомление, проверяет заявку

4. Admin вручную:
   - Скачивает файлы из заявки
   - Создаёт новый проект в autolabs/labs/
   - Пушит файлы + добавляет .gitlab-ci.yml
   - GitLab CI запускается автоматически

5. После успешного билда и scan:
   - Admin одобряет заявку (изменяет статус: approved)
   - Лаба появляется в каталоге платформы
   - Teacher может запускать лабу для студентов
```

#### Вариант B: Автоматизация (будущее)

```
1-3. [Те же шаги]

4. Admin жмёт "Approve" в админ-панели

5. Backend Platform автоматически:
   - Создаёт новую ветку в autolabs/labs/ (через GitLab API)
   - Коммитит файлы из заявки
   - Создаёт .gitlab-ci.yml (из шаблона)
   - Создаёт Merge Request

6. GitLab CI запускается, билдит, сканирует

7. Если pipeline успешен:
   - Backend Platform автоматически мерджит MR
   - Лаба добавляется в каталог
   - Teacher получает уведомление

8. Если pipeline failed:
   - MR остаётся открытым
   - Admin вручную исправляет и мерджит
```

**Рекомендация:** Вариант A для MVP, Вариант B для production.

### Пример: .gitlab-ci.yml для лабораторной работы

```yaml
# autolabs/labs/sql-injection/.gitlab-ci.yml

stages:
  - build
  - scan
  - push

variables:
  IMAGE_NAME: $CI_REGISTRY/autolabs/labs/sql-injection
  IMAGE_TAG: latest

# Stage 1: Build Docker Image
build:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
    - docker save ${IMAGE_NAME}:${IMAGE_TAG} -o lab-image.tar
  artifacts:
    paths:
      - lab-image.tar
    expire_in: 1 hour
  only:
    - main

# Stage 2: Security Scan (только базовые образы)
scan_base_images:
  stage: scan
  image: aquasec/trivy:latest
  script:
    # Сканируем только базовые образы (не уязвимости приложения)
    - trivy image --input lab-image.tar --severity CRITICAL --exit-code 0
    # Примечание: уязвимости в приложении преднамеренные (для обучения)
    # Сканируем только критичные уязвимости в base image (ubuntu, python, etc)
  dependencies:
    - build
  allow_failure: true  # Не блокируем при находках (Admin проверяет вручную)
  only:
    - main

# Stage 3: Push to Registry
push:
  stage: push
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker load -i lab-image.tar
    - docker push ${IMAGE_NAME}:${IMAGE_TAG}
  dependencies:
    - build
  only:
    - main

# Уведомление Admin о новой лабе
notify_admin:
  stage: .post
  image: alpine:latest
  script:
    - apk add --no-cache curl
    - |
      curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d "chat_id=${TELEGRAM_ADMIN_CHAT_ID}" \
        -d "text=✅ New Lab Image Built: ${CI_PROJECT_NAME}\nImage: ${IMAGE_NAME}:${IMAGE_TAG}\nReady for approval in admin panel"
  when: on_success
  only:
    - main
```

---

## Helm Charts структура

### Репозиторий: autolabs/infrastructure/helm-charts

```
helm-charts/
│
├── backend-platform/
│   ├── Chart.yaml
│   ├── values.yaml              # Дефолтные значения
│   ├── values-dev.yaml          # Dev окружение
│   ├── values-prod.yaml         # Prod окружение
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       └── hpa.yaml             # HorizontalPodAutoscaler (опционально)
│
├── celery-workers/
│   └── [аналогичная структура]
│
├── frontend/
│   └── [аналогичная структура]
│
├── zitadel/
│   └── [аналогичная структура]
│
└── rabbitmq/
    └── [аналогичная структура]
```

### Пример: values.yaml для backend-platform

```yaml
# helm-charts/backend-platform/values.yaml

replicaCount: 2

image:
  repository: registry.gitlab.local/autolabs/platform/backend-platform
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 8000

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: backend-secrets
        key: database_url

  - name: REDIS_URL
    value: "redis://redis.data.svc.cluster.local:6379"

  - name: MINIO_ENDPOINT
    value: "minio.data.svc.cluster.local:9000"

  - name: RABBITMQ_HOST
    value: "rabbitmq.messaging.svc.cluster.local"

  - name: ZITADEL_URL
    value: "https://auth.autolabs.local"

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Пример: values-dev.yaml (переопределение для dev)

```yaml
# helm-charts/backend-platform/values-dev.yaml

replicaCount: 1

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

env:
  - name: DEBUG
    value: "true"

  - name: LOG_LEVEL
    value: "DEBUG"
```

### Деплой с разными окружениями

```bash
# Dev окружение
helm upgrade --install backend-platform ./helm-charts/backend-platform \
  --namespace platform \
  -f ./helm-charts/backend-platform/values.yaml \
  -f ./helm-charts/backend-platform/values-dev.yaml

# Prod окружение
helm upgrade --install backend-platform ./helm-charts/backend-platform \
  --namespace platform \
  -f ./helm-charts/backend-platform/values.yaml \
  -f ./helm-charts/backend-platform/values-prod.yaml
```

---

## GitLab Runner Setup

### Установка на VM GitLab

**GitLab Runner** развёрнут на той же VM что и GitLab для упрощения.

```bash
# Установка GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Регистрация Runner
sudo gitlab-runner register \
  --url https://registry.gitlab.local \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --docker-image docker:24.0 \
  --docker-privileged \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock

# Настройка concurrency
sudo nano /etc/gitlab-runner/config.toml
# concurrent = 4  (количество параллельных jobs)

# Перезапуск
sudo gitlab-runner restart
```

### Executor: Docker

**Почему Docker executor:**
- ✅ Изоляция каждого job в отдельном контейнере
- ✅ Поддержка Docker-in-Docker (для билда образов)
- ✅ Простота настройки

**Альтернатива (будущее):** Kubernetes executor (Runner внутри K8s, jobs как pods)

---

## Security Scanning

### Trivy для сканирования образов

**Trivy** - open-source сканер уязвимостей от Aqua Security.

#### Для platform образов

```yaml
scan:
  script:
    - trivy image --input image.tar --severity HIGH,CRITICAL --exit-code 1
    # exit-code 1: pipeline падает при HIGH/CRITICAL уязвимостях
```

**Политика:**
- **CRITICAL** уязвимости → блокируем деплой
- **HIGH** уязвимости → warning, но разрешаем деплой (Admin решает)
- **MEDIUM/LOW** → игнорируем

#### Для labs образов

```yaml
scan_base_images:
  script:
    - trivy image --input lab-image.tar --severity CRITICAL --exit-code 0
    # Сканируем только базовые образы (ubuntu, python, mysql)
    # Уязвимости в приложении преднамеренные (для обучения)
```

**Политика:**
- Сканируем только **базовые образы** (FROM ubuntu:22.04, FROM python:3.11)
- Уязвимости в **приложении лабы** игнорируем (они и есть цель обучения)
- Admin вручную проверяет report

### Trivy report в Telegram

```yaml
scan:
  after_script:
    - trivy image --input image.tar --format json -o trivy-report.json
    - |
      # Отправка критичных уязвимостей в Telegram
      CRITICAL_COUNT=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' trivy-report.json)
      if [ "$CRITICAL_COUNT" -gt 0 ]; then
        curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
          -d "chat_id=${TELEGRAM_ADMIN_CHAT_ID}" \
          -d "text=⚠️ CRITICAL vulnerabilities found: ${CRITICAL_COUNT}\nProject: ${CI_PROJECT_NAME}\nPipeline: ${CI_PIPELINE_URL}"
      fi
```

---

## Уведомления в Telegram

### Настройка Telegram бота

```bash
# 1. Создать бота через @BotFather
# 2. Получить токен: 123456789:ABCdefGHIjklMNOpqrsTUVwxyz
# 3. Создать группу для уведомлений
# 4. Добавить бота в группу
# 5. Получить chat_id группы:
curl https://api.telegram.org/bot<TOKEN>/getUpdates
```

### События для уведомлений

| Событие | Приоритет | Кому |
|---------|-----------|------|
| Pipeline FAILED (platform) | 🔴 HIGH | Разработчики + Admin |
| Deploy SUCCESS (platform) | ✅ INFO | Разработчики |
| New Lab Image Built | ✅ INFO | Admin |
| CRITICAL vulnerabilities | ⚠️ HIGH | Admin |
| Deploy FAILED | 🔴 HIGH | DevOps + Admin |

### Пример сообщения в Telegram

```
✅ Deploy SUCCESS: backend-platform → K8s
Version: latest
Commit: abc1234
Author: Ivan Ivanov
Branch: main
Time: 2025-12-01 15:30:00
```

---

## Workflow: От Commit до Production

### Сценарий 1: Обновление Backend Platform

```
1. Developer: git push origin feature/add-new-api

2. GitLab CI (feature branch):
   ├── Build ✅
   ├── Scan ✅
   └── Push ✅ (тег: feature-add-new-api)

3. Developer: Создаёт Merge Request → main

4. Code Review → Approve → Merge

5. GitLab CI (main branch):
   ├── Build ✅
   ├── Scan ✅
   ├── Push ✅ (тег: latest)
   └── Deploy (manual) → Ожидание запуска

6. DevOps/Admin: Запускает Deploy job вручную

7. GitLab CI:
   ├── Clone helm-charts
   ├── helm upgrade backend-platform
   └── Telegram notification ✅

8. Backend Platform обновлён в K8s namespace: platform
```

### Сценарий 2: Публикация новой лабораторной работы

```
1. Teacher: Создаёт заявку на лабу "SQL Injection v2"
   - Загружает lab-config.yaml
   - Загружает Dockerfile
   - Загружает код приложения

2. Backend Platform: Сохраняет заявку (статус: pending)

3. Admin: Получает Telegram уведомление

4. Admin: Проверяет заявку в админ-панели
   - Просматривает Dockerfile (нет вредоносного кода)
   - Просматривает lab-config.yaml (корректная конфигурация)
   - Проверяет описание лабы

5. Admin: Скачивает файлы, создаёт проект в GitLab
   - git clone autolabs/labs/sql-injection-v2
   - Добавляет файлы из заявки
   - Добавляет .gitlab-ci.yml (из шаблона)
   - git push origin main

6. GitLab CI:
   ├── Build ✅
   ├── Scan (базовые образы) ✅
   └── Push ✅ (registry.gitlab.local/autolabs/labs/sql-injection-v2:latest)

7. GitLab CI: Telegram notification Admin

8. Admin: Одобряет заявку (POST /api/v1/labs/requests/123/approve)

9. Backend Platform:
   - Изменяет статус заявки: approved
   - Добавляет лабу в каталог платформы
   - Отправляет уведомление Teacher

10. Teacher: Видит новую лабу, может запускать для студентов
```

---

## Документация в этой секции

| Файл | Описание |
|------|----------|
| `README.md` | Общий обзор GitLab CI/CD (этот файл) |
| `platform-pipelines.md` | Детальное описание пайплайнов для platform образов |
| `lab-pipelines.md` | Детальное описание пайплайнов для labs образов |
| `helm-charts-structure.md` | Структура Helm charts репозитория |
| `gitlab-runner-setup.md` | Установка и настройка GitLab Runner |
| `security-scanning.md` | Trivy сканирование и политики безопасности |
| `telegram-notifications.md` | Настройка Telegram уведомлений |
| `cicd-flow.drawio` | Схема CI/CD потока (визуализация) |

---

## Будущие улучшения (вне MVP)

### GitOps с ArgoCD

**Проблема текущего подхода:**
- Деплой через GitLab CI (push-based)
- Нужен доступ к K8s из GitLab Runner

**Решение: ArgoCD (pull-based GitOps)**
```
1. Developer пушит код → GitLab CI билдит образ
2. GitLab CI обновляет image.tag в helm-charts репозитории
3. ArgoCD в K8s автоматически синхронизирует изменения
4. Нет нужды в KUBECONFIG в GitLab
```

### Multi-environment с отдельными namespace

```
namespace: platform-dev
namespace: platform-stage
namespace: platform-prod
```

### Auto-scaling GitLab Runner

- Kubernetes executor вместо Docker
- Runner внутри K8s, jobs как ephemeral pods
- Автоматическое масштабирование по нагрузке

### Canary Deployments

- Постепенный раскат новых версий
- 10% трафика → новая версия
- Мониторинг метрик → полный раскат или rollback

### Автоматизация заявок на лабы (Вариант B)

- Backend Platform автоматически создаёт MR в GitLab
- Admin только проверяет и мерджит
- Полная автоматизация от заявки до production

---

## Связь с другими компонентами

- **[06-authorization](../06-authorization/README.md)** - Права доступа к GitLab группам
- **[02-backend-platform](../02-backend-platform/)** - Обработка заявок на лабы
- **[03-celery-workers](../03-celery-workers/)** - Деплой после билда образов
- **[07-lab-deployments](../07-lab-deployments/)** - Детальная структура lab-config.yaml

---

**Версия:** 1.0
**Дата последнего обновления:** 2025-12-01
