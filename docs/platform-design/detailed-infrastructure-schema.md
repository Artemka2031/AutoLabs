# Детальная схема инфраструктуры AutoLabs

## Дата создания
2025-11-29

---

## Общая архитектура

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROXMOX HYPERVISOR                                 │
│                     (Виртуализация на базе KVM/LXC)                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────────┐
│   VM: GitLab  │          │  VM: Database │          │ VM: K8s Cluster   │
│  4 CPU, 8GB   │          │  4 CPU, 8GB   │          │ 12 CPU, 16GB      │
└───────────────┘          └───────────────┘          └───────────────────┘
        │                           │                           │
        ▼                           ▼                           ▼
  CI/CD + Registry         PostgreSQL + Redis           Приложения платформы
                              + Minio                    + Лабораторные работы
```

---

## 1. VM: GitLab (CI/CD и Container Registry)

### Технические характеристики
- **CPU:** 4 cores
- **RAM:** 8 GB
- **Диск:** 100 GB SSD
- **ОС:** Ubuntu 22.04 LTS / Debian 12

### Назначение
Централизованная система для управления кодом, CI/CD пайплайнами и хранения Docker образов.

### Компоненты

#### 1.1 GitLab CE/EE
**Что делает:**
- Хранение Git репозиториев (код платформы, конфигурации лабораторных работ)
- Web интерфейс для разработки
- Управление пользователями и доступом

**Репозитории:**
```
autolabs/
├── frontend/              # React/Vue приложение
├── backend-platform/      # FastAPI сервис для платформы
├── backend-infrastructure/# FastAPI сервис для K8s управления
├── lab-configs/           # YAML конфигурации лабораторных работ
├── infrastructure/        # Terraform/Helm конфигурации
└── docs/                  # Документация
```

#### 1.2 GitLab CI/CD
**Что делает:**
- Автоматическая сборка Docker образов при push в репозиторий
- Запуск тестов (pytest, jest)
- Деплой приложений в K8s кластер

**Пример пайплайна:**
```
Git Push → GitLab CI → Build Docker Image → Push to Registry → Deploy to K8s
```

#### 1.3 GitLab Container Registry
**Что делает:**
- Хранение Docker образов приложений платформы
- Хранение Docker образов лабораторных работ

**Примеры образов:**
```
registry.gitlab.local/autolabs/frontend:latest
registry.gitlab.local/autolabs/backend-platform:v1.2.0
registry.gitlab.local/autolabs/labs/sql-injection:v1.0
registry.gitlab.local/autolabs/labs/network-analysis:v2.1
```

#### 1.4 GitLab Runner
**Что делает:**
- Выполняет CI/CD задачи
- Билдит Docker образы
- Деплоит в K8s через kubectl/helm

**Развёртывание:**
- Docker executor (запуск задач в контейнерах)
- Доступ к K8s API для деплоя

### Сетевые подключения
- **Порт 443 (HTTPS):** Веб-интерфейс, Git операции
- **Порт 5050 (Registry):** Docker Registry
- **→ K8s API:** Деплой приложений
- **→ VM Database:** Собственная БД GitLab (PostgreSQL)

---

## 2. VM: Database (Централизованное хранилище данных)

### Технические характеристики
- **CPU:** 4 cores
- **RAM:** 8 GB
- **Диск:** 200 GB SSD (можно расширить для Minio)
- **ОС:** Ubuntu 22.04 LTS / Debian 12

### Назначение
Централизованное хранилище данных для всех компонентов платформы. Вынесено из K8s для независимости, производительности и упрощения backup.

### Компоненты

#### 2.1 PostgreSQL (Реляционная база данных)
**Что делает:**
- Хранит метаданные платформы (пользователи, курсы, задания, группы)
- Хранит метаданные лабораторных работ (конфигурации, статусы)
- Хранит данные GitLab (репозитории, users, CI/CD jobs)
- Хранит данные Zitadel (пользователи, роли, сессии)

**Базы данных:**
```
PostgreSQL Instance
├── autolabs_platform      # Данные платформы
│   ├── users              # Пользователи платформы
│   ├── courses            # Курсы
│   ├── assignments        # Задания
│   ├── student_groups     # Группы студентов
│   └── submissions        # Результаты выполнения
│
├── autolabs_infrastructure # Данные инфраструктуры
│   ├── lab_deployments    # Активные развёртывания лаб
│   ├── lab_configs        # Конфигурации лаб
│   ├── resource_quotas    # Квоты ресурсов
│   └── audit_logs         # Логи операций
│
├── gitlab                 # База данных GitLab
│
└── zitadel                # База данных Zitadel
```

**Настройки:**
- PostgreSQL 15
- Автоматический backup (pg_dump ежедневно)
- Репликация (опционально для production)

#### 2.2 Redis (In-Memory кэш и очереди)
**Что делает:**
- Кэширование API ответов (уменьшение нагрузки на PostgreSQL)
- Хранение пользовательских сессий
- Celery broker (альтернатива RabbitMQ для некоторых задач)
- Rate limiting (ограничение запросов к API)

**Использование:**
```
Redis Instance
├── DB 0: API кэш (TTL: 5-60 минут)
├── DB 1: Пользовательские сессии (TTL: 24 часа)
├── DB 2: Celery tasks (если используется вместо RabbitMQ)
└── DB 3: Rate limiting counters
```

**Настройки:**
- Redis 7
- Persistence: RDB + AOF (сохранение на диск)
- Maxmemory policy: allkeys-lru

#### 2.3 Minio (S3-совместимое объектное хранилище)
**Что делает:**
- Хранит конфигурации лабораторных работ (YAML файлы, скрипты)
- Хранит статичные файлы (изображения, документы курсов)
- Хранит архивы volumes студентов (для переиспользования лаб)
- Хранит backup данных

**Структура buckets:**
```
Minio Instance
├── lab-configs/           # YAML конфигурации лаб
│   ├── sql-injection.yaml
│   ├── network-pentest.yaml
│   └── ...
│
├── lab-volumes/           # Архивы volumes студентов
│   ├── lab-123-student-456.tar.gz
│   └── ...
│
├── static-files/          # Статичные файлы платформы
│   ├── course-images/
│   ├── assignments-files/
│   └── ...
│
└── backups/               # Backup данных
    ├── postgres-dumps/
    └── gitlab-backups/
```

**Настройки:**
- Minio latest
- Versioning: включено (для критичных buckets)
- Access: приватный (доступ только через API keys)

### Сетевые подключения
- **PostgreSQL :5432** ← Backend Platform, Backend Infrastructure, Zitadel, GitLab
- **Redis :6379** ← Backend Platform, Backend Infrastructure
- **Minio :9000 (API), :9001 (Console)** ← Backend Platform, Backend Infrastructure

### Backup стратегия
- **PostgreSQL:** pg_dump каждую ночь → Minio bucket backups/
- **Redis:** RDB snapshots каждый час
- **Minio:** Репликация на внешний S3 (опционально)
- **Proxmox:** VM snapshots еженедельно

---

## 3. VM: Kubernetes Cluster (Оркестрация приложений)

### Технические характеристики
- **CPU:** 12 cores (распределено между нодами)
- **RAM:** 16 GB (распределено между нодами)
- **Диск:** 100 GB SSD
- **ОС:** Ubuntu 22.04 LTS

### Архитектура кластера

#### Вариант A: Single-Node (MVP, ограниченные ресурсы)
```
VM: K8s-All-in-One (12 CPU, 16 GB)
├── Control Plane (2 CPU, 2 GB)
│   ├── kube-apiserver
│   ├── kube-scheduler
│   ├── kube-controller-manager
│   └── etcd
│
└── Worker Node (10 CPU, 14 GB)
    └── Поды приложений
```

#### Вариант B: Multi-Node (Production, больше ресурсов)
```
VM-1: K8s-Control-Plane (4 CPU, 4 GB)
├── kube-apiserver
├── kube-scheduler
├── kube-controller-manager
└── etcd

VM-2: K8s-Worker-1 (4 CPU, 6 GB)
└── Поды приложений (платформа, инфраструктура)

VM-3: K8s-Worker-2 (4 CPU, 6 GB)
└── Поды приложений (лабораторные работы)
```

### Назначение
Оркестрация контейнеров приложений платформы и динамических лабораторных работ студентов.

---

## 4. Kubernetes Namespace структура

### 4.1 Namespace: platform

**Назначение:** Платформенные сервисы для взаимодействия пользователей

#### Компоненты:

##### 4.1.1 Frontend (React/Vue SPA)
**Deployment:** 1-2 replicas
**Ресурсы:** 200m CPU, 256Mi RAM per pod
**Образ:** `registry.gitlab.local/autolabs/frontend:latest`

**Что делает:**
- Веб-интерфейс для студентов, преподавателей, администраторов
- Отображение курсов, заданий, лабораторных работ
- Панель управления для преподавателей
- Мониторинг статуса лабораторных работ

**Технологии:** React/Vue.js, Nginx (статика)

##### 4.1.2 Backend Platform (FastAPI)
**Deployment:** 2-3 replicas
**Ресурсы:** 500m CPU, 512Mi RAM per pod
**Образ:** `registry.gitlab.local/autolabs/backend-platform:latest`

**Что делает:**
- REST API для Frontend
- CRUD операции: курсы, студенты, группы, задания
- Валидация данных пользователей
- Интеграция с Zitadel (проверка JWT токенов)
- Отправка запросов в Backend Infrastructure для запуска лаб
- Работа с PostgreSQL (данные платформы)
- Работа с Redis (кэш, сессии)
- Работа с Minio (загрузка/скачивание файлов)

**Технологии:** FastAPI (Python 3.11), SQLAlchemy, Pydantic

**API Endpoints (примеры):**
```
GET  /api/v1/courses           # Список курсов
POST /api/v1/courses           # Создать курс
GET  /api/v1/labs              # Список лабораторных работ
POST /api/v1/labs/{id}/start   # Запустить лабу (→ Backend Infra)
GET  /api/v1/students/{id}/progress  # Прогресс студента
```

**Service:** ClusterIP, порт 8000

---

### 4.2 Namespace: infrastructure

**Назначение:** Управление K8s ресурсами и жизненным циклом лабораторных работ

#### Компоненты:

##### 4.2.1 Backend Infrastructure (FastAPI)
**Deployment:** 1-2 replicas
**Ресурсы:** 500m CPU, 512Mi RAM per pod
**Образ:** `registry.gitlab.local/autolabs/backend-infrastructure:latest`

**Что делает:**
- Получает запросы от Backend Platform на создание лабораторных работ
- Управляет K8s API через Python Kubernetes Client
- Создаёт/удаляет Pods, Services, Network Policies в namespace: labs
- Управляет PersistentVolumeClaims для студентов
- Архивирует volumes в Minio при завершении лаб
- Отправляет статусы и учётные данные в RabbitMQ
- Логирует все операции в PostgreSQL

**Технологии:** FastAPI, Python Kubernetes Client, asyncio

**Основные функции:**
```
create_lab_deployment(lab_id, students, config)
  ├── Парсит YAML конфигурацию из Minio
  ├── Создаёт Pods для каждого студента/группы
  ├── Создаёт Services (SSH доступ)
  ├── Создаёт Network Policies (изоляция)
  ├── Создаёт PVC (persistent storage)
  └── Отправляет credentials в RabbitMQ

delete_lab_deployment(lab_id)
  ├── Архивирует volumes в Minio
  ├── Удаляет Pods, Services, Network Policies
  └── Обновляет статус в PostgreSQL
```

**Service:** ClusterIP, порт 8000

##### 4.2.2 Celery Worker
**Deployment:** 2-3 replicas
**Ресурсы:** 300m CPU, 384Mi RAM per pod
**Образ:** `registry.gitlab.local/autolabs/backend-infrastructure:latest` (тот же, другой entrypoint)

**Что делает:**
- Обрабатывает асинхронные задачи из RabbitMQ
- Создание лабораторных работ (долгие операции)
- Архивация volumes
- Периодические задачи (cleanup старых лаб, мониторинг)

**Технологии:** Celery, RabbitMQ broker

**Периодические задачи:**
```
@celery.beat(cron="0 2 * * *")  # Каждую ночь в 2:00
def cleanup_old_labs():
    # Удаление лаб старше 7 дней

@celery.beat(cron="*/5 * * * *")  # Каждые 5 минут
def archive_finished_volumes():
    # Архивация volumes завершённых лаб
```

---

### 4.3 Namespace: auth

**Назначение:** Централизованная авторизация и аутентификация

#### Компоненты:

##### 4.3.1 Zitadel (Identity and Access Management)
**Deployment:** 2 replicas
**Ресурсы:** 500m CPU, 512Mi RAM per pod
**Образ:** `ghcr.io/zitadel/zitadel:latest`

**Что делает:**
- Управление пользователями (students, teachers, admins)
- Аутентификация (логин/логаут)
- Выдача JWT токенов (OAuth2 / OIDC)
- Управление ролями и permissions (RBAC)
- Multi-factor authentication (опционально)

**Технологии:** Zitadel (Go), PostgreSQL backend

**Роли в системе:**
```
Roles:
├── admin           # Полный доступ ко всей платформе
├── teacher         # Создание курсов, управление лабами
├── student         # Просмотр курсов, выполнение лаб
└── viewer          # Только чтение
```

**Интеграция:**
```
User → Frontend → Zitadel (логин) → JWT token
Frontend → Backend Platform (запрос + JWT token)
Backend Platform → Zitadel API (валидация токена) → Разрешить/Запретить
```

**Service:** ClusterIP, порт 8080
**Ingress:** `https://autolabs.local/auth` (внешний доступ для логина)

---

### 4.4 Namespace: messaging

**Назначение:** Асинхронная коммуникация между сервисами

#### Компоненты:

##### 4.4.1 RabbitMQ (Message Broker)
**StatefulSet:** 1 replica (можно кластер для HA)
**Ресурсы:** 500m CPU, 512Mi RAM per pod
**Образ:** `rabbitmq:3.12-management`

**Что делает:**
- Очередь задач для Celery Workers
- Передача сообщений между Backend Platform ↔ Backend Infrastructure
- Отправка уведомлений (статусы лаб, credentials для студентов)
- Event-driven архитектура

**Exchanges и Queues:**
```
RabbitMQ
├── Exchange: lab.events
│   ├── Queue: lab.create      # Создание лабораторных работ
│   ├── Queue: lab.delete      # Удаление лабораторных работ
│   └── Queue: lab.status      # Обновления статусов
│
├── Exchange: notifications
│   ├── Queue: student.notifications  # Уведомления студентам
│   └── Queue: teacher.notifications  # Уведомления преподавателям
│
└── Exchange: celery
    └── Queue: celery.tasks    # Celery worker tasks
```

**Service:** ClusterIP, порты 5672 (AMQP), 15672 (Management UI)

---

### 4.5 Namespace: labs

**Назначение:** Динамические лабораторные работы для студентов

**Особенность:** Ресурсы создаются и удаляются автоматически Backend Infrastructure

#### Типичные ресурсы (создаются динамически):

##### 4.5.1 Pod: Лабораторная работа студента
**Пример:** `lab-123-student-456` (уникальное имя)
**Ресурсы:** 500m-1000m CPU, 512Mi-1Gi RAM (зависит от конфигурации лабы)
**Образ:** Определяется конфигурацией (например, `registry.gitlab.local/autolabs/labs/sql-injection:v1`)

**Что внутри пода:**
- Контейнер(ы) с уязвимым приложением / инструментами
- SSH сервер (для доступа студента)
- Инициализационные скрипты

**Labels:**
```yaml
labels:
  app: lab
  lab_id: "123"
  student_id: "456"
  course_id: "cybersec-101"
  deployment_mode: "per_student"
```

##### 4.5.2 Service: SSH доступ к лабораторной работе
**Тип:** NodePort (или LoadBalancer в prod)
**Порт:** 22 (SSH)
**NodePort:** 30000-32767 (динамически назначается)

**Как студент подключается:**
```
ssh student456@<k8s-node-ip> -p 30123
# Или через доменное имя:
ssh student456@labs.autolabs.local -p 30123
```

##### 4.5.3 NetworkPolicy: Изоляция лабораторной работы
**Что делает:**
- Изолирует лабу студента от других студентов
- Разрешает/запрещает доступ в интернет (по конфигурации)
- Разрешает доступ только к необходимым сервисам

**Пример политики (запрет интернета):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lab-123-student-456-policy
  namespace: labs
spec:
  podSelector:
    matchLabels:
      lab_id: "123"
      student_id: "456"
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector: {}  # Только внутри namespace
```

##### 4.5.4 PersistentVolumeClaim: Хранилище данных студента
**Размер:** 5Gi (по умолчанию, настраивается в конфигурации лабы)
**AccessMode:** ReadWriteOnce
**StorageClass:** local-path / ceph / nfs (зависит от K8s setup)

**Что хранится:**
- Файлы, созданные студентом во время работы
- Изменения в конфигурациях
- Результаты выполнения заданий

**Lifecycle:**
- Создаётся при запуске лабы
- Сохраняется при остановке лабы (7 дней)
- Архивируется в Minio через 7 дней (tar.gz)
- Удаляется PVC после архивации

---

### 4.6 Namespace: ingress-nginx

**Назначение:** Внутренний HTTP/HTTPS маршрутизатор для K8s сервисов

#### Компоненты:

##### 4.6.1 Nginx Ingress Controller
**Deployment:** 2 replicas (HA)
**Ресурсы:** 200m CPU, 256Mi RAM per pod
**Образ:** `registry.k8s.io/ingress-nginx/controller:latest`

**Что делает:**
- Маршрутизация HTTP/HTTPS трафика внутри кластера
- Терминация SSL (если не делается на внешнем Nginx)
- Load balancing между репликами сервисов
- WebSocket support (для real-time уведомлений)

**Маршруты (Ingress Resources):**
```
https://autolabs.local/          → Frontend (namespace: platform)
https://autolabs.local/api/      → Backend Platform (namespace: platform)
https://autolabs.local/auth/     → Zitadel (namespace: auth)
https://autolabs.local/infra/    → Backend Infrastructure (namespace: infrastructure)
```

**Service:** LoadBalancer / NodePort (в зависимости от setup)

---

## 5. VM: Nginx (Внешний Reverse Proxy) - Опционально

### Технические характеристики
- **CPU:** 1 core
- **RAM:** 1 GB
- **Диск:** 20 GB
- **ОС:** Ubuntu 22.04 LTS

### Назначение
Внешний шлюз для SSL termination и первичной маршрутизации.

### Компоненты

#### 5.1 Nginx (Bare-metal)
**Что делает:**
- SSL/TLS termination (Let's Encrypt сертификаты)
- Первичная маршрутизация на K8s Ingress Controller
- Rate limiting (защита от DDoS)
- Логирование внешних запросов

**Конфигурация:**
```
Пользователь (HTTPS :443)
    ↓
Nginx (SSL termination)
    ↓
Nginx Ingress Controller (K8s) (HTTP :80)
    ↓
Frontend / Backend / Zitadel
```

**Альтернатива:** Можно использовать только Nginx Ingress Controller в K8s с cert-manager для Let's Encrypt.

---

## 6. Инструменты управления инфраструктурой

### 6.1 Terraform (Infrastructure as Code)

**Назначение:** Создание и управление виртуальными машинами в Proxmox

**Что создаёт Terraform:**
```
terraform apply
    ↓
Proxmox API
    ↓
Создаёт VMs:
├── VM GitLab (4 CPU, 8 GB, 100 GB disk)
├── VM Database (4 CPU, 8 GB, 200 GB disk)
├── VM K8s-Node-1 (4 CPU, 6 GB, 100 GB disk)
├── VM K8s-Node-2 (4 CPU, 5 GB, 100 GB disk)
└── VM K8s-Node-3 (4 CPU, 5 GB, 100 GB disk)
    ↓
Настраивает сети, firewall
```

**Структура конфигурации:**
```
terraform/
├── providers.tf           # Proxmox provider
├── variables.tf           # Переменные (CPU, RAM, IP адреса)
├── vms.tf                 # Определение VMs
├── networks.tf            # Сетевая конфигурация
├── outputs.tf             # Вывод IP адресов после создания
└── environments/
    ├── dev.tfvars         # Dev окружение (минимальные ресурсы)
    ├── stage.tfvars       # Stage окружение
    └── prod.tfvars        # Prod окружение (полные ресурсы)
```

**Multi-environment deployment:**
```bash
# Развернуть dev окружение
terraform apply -var-file=environments/dev.tfvars

# Развернуть prod окружение
terraform apply -var-file=environments/prod.tfvars
```

**Преимущества:**
- Воспроизводимость (одинаковая инфраструктура каждый раз)
- Версионирование (инфраструктура в Git)
- Multi-cloud готовность (легко перенести на Yandex Cloud, AWS, etc)

---

### 6.2 Helm (Пакетный менеджер для Kubernetes)

**Назначение:** Установка статичных компонентов в K8s

**Что устанавливает Helm:**

#### Готовые Charts (из репозиториев):
```bash
# Добавление Bitnami репозитория (если данные в K8s)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Установка (если БД внутри K8s - альтернативный вариант)
# Для нашей архитектуры БД на отдельной VM, поэтому не используем
```

#### Кастомные Charts (для приложений платформы):
```
helm/
├── autolabs-platform/     # Chart для platform namespace
│   ├── Chart.yaml
│   ├── values.yaml        # Дефолтные значения
│   ├── values-dev.yaml    # Dev окружение
│   ├── values-prod.yaml   # Prod окружение
│   └── templates/
│       ├── frontend-deployment.yaml
│       ├── backend-platform-deployment.yaml
│       ├── services.yaml
│       └── ingress.yaml
│
├── autolabs-infrastructure/  # Chart для infrastructure namespace
│   └── ...
│
├── zitadel/               # Chart для Zitadel
│   └── ...
│
└── rabbitmq/              # Chart для RabbitMQ
    └── ...
```

**Развёртывание:**
```bash
# Установка всех компонентов платформы
helm install autolabs-platform ./helm/autolabs-platform -f values-dev.yaml

# Обновление конфигурации
helm upgrade autolabs-platform ./helm/autolabs-platform -f values-prod.yaml

# Откат при проблемах
helm rollback autolabs-platform 1
```

---

### 6.3 Python Kubernetes Client (Динамическое управление)

**Назначение:** Создание/удаление лабораторных работ через Backend Infrastructure

**Интеграция:**
```
Backend Infrastructure (FastAPI)
    ↓
Python Kubernetes Client
    ↓
K8s API Server
    ↓
Создаёт в namespace: labs
├── Pod (lab-123-student-456)
├── Service (SSH доступ)
├── NetworkPolicy (изоляция)
└── PVC (хранилище)
```

**Пример использования:**
```python
# backend-infrastructure/app/k8s_manager.py
from kubernetes import client, config

def create_student_lab(lab_id: str, student_id: str, lab_config: dict):
    # Загрузка конфигурации K8s
    config.load_incluster_config()

    # Создание API клиента
    v1 = client.CoreV1Api()

    # Создание Pod
    pod = create_pod_manifest(lab_id, student_id, lab_config)
    v1.create_namespaced_pod(namespace="labs", body=pod)

    # Создание Service
    service = create_service_manifest(lab_id, student_id)
    v1.create_namespaced_service(namespace="labs", body=service)

    # Создание NetworkPolicy
    policy = create_network_policy(lab_id, student_id, lab_config)
    networking_v1.create_namespaced_network_policy(namespace="labs", body=policy)
```

---

## 7. Сетевая архитектура

### 7.1 Внешний доступ

```
Интернет
    ↓
[DNS: autolabs.local]
    ↓
[Внешний IP сервера]
    ↓
[Nginx VM] :443 (SSL termination)
    ↓
[Nginx Ingress Controller K8s] :80
    ↓
┌──────────────┬──────────────┬──────────────┐
│  Frontend    │  Backend     │  Zitadel     │
│  :3000       │  :8000       │  :8080       │
└──────────────┴──────────────┴──────────────┘
```

### 7.2 Внутренняя коммуникация

```
Backend Platform (namespace: platform)
    ↓
├── → PostgreSQL VM :5432 (метаданные платформы)
├── → Redis VM :6379 (кэш, сессии)
├── → Minio VM :9000 (файлы)
├── → Zitadel (namespace: auth) :8080 (валидация токенов)
└── → Backend Infrastructure (namespace: infrastructure) :8000 (запросы на создание лаб)

Backend Infrastructure (namespace: infrastructure)
    ↓
├── → K8s API (создание ресурсов в namespace: labs)
├── → PostgreSQL VM :5432 (метаданные инфраструктуры)
├── → Minio VM :9000 (конфигурации лаб, архивы volumes)
└── → RabbitMQ (namespace: messaging) :5672 (очереди задач)

Celery Worker (namespace: infrastructure)
    ↓
├── → RabbitMQ (namespace: messaging) :5672 (получение задач)
├── → K8s API (выполнение операций)
└── → Minio VM :9000 (архивация volumes)
```

### 7.3 Изоляция лабораторных работ

```
namespace: labs
├── Lab 1 (SQL Injection)
│   ├── Pod: lab-1-student-alice
│   ├── NetworkPolicy: запрет всех egress кроме localhost
│   └── Изолирован от Lab 2, Lab 3, ...
│
├── Lab 2 (Network Pentest)
│   ├── Pod: lab-2-group-cybersec-101
│   ├── NetworkPolicy: разрешён доступ в интернет
│   └── Изолирован от Lab 1, Lab 3, ...
│
└── Lab 3 (Multi-container)
    ├── Pod: lab-3-student-bob-webapp
    ├── Pod: lab-3-student-bob-database
    ├── Service: internal communication between pods
    ├── NetworkPolicy: запрет доступа из других лаб
    └── Изолирован от Lab 1, Lab 2, ...
```

---

## 8. Процесс развёртывания лабораторной работы

### Полный цикл (от преподавателя до студента):

```
1. Преподаватель создаёт конфигурацию лабораторной работы
   ├── Пишет Dockerfile(s) для контейнеров
   ├── Push в GitLab → GitLab CI билдит образ → Push в GitLab Registry
   └── Создаёт YAML конфигурацию (lab-config.yaml)

2. Преподаватель загружает конфигурацию через Frontend
   ↓
   Frontend → Backend Platform (POST /api/v1/labs)
   ↓
   Backend Platform → Minio (сохранение lab-config.yaml в bucket: lab-configs/)
   ↓
   Backend Platform → PostgreSQL (сохранение метаданных: lab_id, название, описание)

3. Преподаватель запускает лабораторную работу для группы студентов
   ↓
   Frontend → Backend Platform (POST /api/v1/labs/{id}/start, body: {student_ids: [...]})
   ↓
   Backend Platform → Backend Infrastructure (POST /infra/v1/deploy, body: {lab_id, student_ids})
   ↓
   Backend Infrastructure → RabbitMQ (отправка задач в очередь lab.create)

4. Celery Worker обрабатывает задачи
   ↓
   Celery Worker → RabbitMQ (получение задачи)
   ↓
   Celery Worker → Minio (скачивание lab-config.yaml)
   ↓
   Celery Worker → K8s API (создание ресурсов для каждого студента):
       ├── Создание Pod (образ из GitLab Registry)
       ├── Создание Service (NodePort для SSH)
       ├── Создание NetworkPolicy (изоляция)
       └── Создание PVC (persistent storage)
   ↓
   Celery Worker → RabbitMQ (отправка уведомления в очередь student.notifications)
       └── Payload: {student_id, ssh_host, ssh_port, ssh_credentials}

5. Backend Platform получает уведомления
   ↓
   Backend Platform → PostgreSQL (обновление статуса: "running")
   ↓
   Backend Platform → WebSocket → Frontend (real-time обновление для преподавателя)

6. Студент получает доступ к лабораторной работе
   ↓
   Студент → Frontend (видит уведомление "Лаба доступна")
   ↓
   Студент → Получает SSH credentials (host, port, password)
   ↓
   Студент → SSH клиент (подключение к лабораторной работе)
       ssh student123@labs.autolabs.local -p 30456
   ↓
   Студент работает в изолированном окружении

7. Завершение лабораторной работы
   ↓
   Преподаватель → Frontend (POST /api/v1/labs/{id}/stop)
   ↓
   Backend Platform → Backend Infrastructure (POST /infra/v1/delete)
   ↓
   Backend Infrastructure → Celery Worker (задача в очередь lab.delete)
   ↓
   Celery Worker:
       ├── K8s API: извлечение PVC данных студента
       ├── Архивация volume → tar.gz
       ├── Minio: загрузка архива в bucket: lab-volumes/
       ├── K8s API: удаление Pod, Service, NetworkPolicy, PVC
       └── PostgreSQL: обновление статуса "stopped"
```

---

## 9. Масштабирование по фазам

### Фаза 1: MVP (Минимальные ресурсы)

**Цель:** Запуск платформы для тестирования и первых пользователей

**Инфраструктура:**
```
Proxmox
├── VM GitLab (4 CPU, 8 GB) → Арендованный VPS Hetzner €6/мес
├── VM Database (4 CPU, 8 GB) → Локальный Proxmox
└── VM K8s Single-Node (12 CPU, 16 GB) → Локальный Proxmox
```

**Ограничения:**
- До 50 одновременных студентов
- До 10 активных лабораторных работ
- Single-point-of-failure (нет HA)

**Инструменты:**
- Terraform: опционально (можно создать VMs вручную)
- Helm: для установки компонентов в K8s
- Python K8s Client: для лабораторных работ

---

### Фаза 2: Production (Средние ресурсы)

**Цель:** Полноценное использование в учебном заведении

**Инфраструктура:**
```
Proxmox
├── VM GitLab (4 CPU, 8 GB)
├── VM Database (8 CPU, 16 GB) → Увеличение для БД
├── VM K8s-Control-Plane (4 CPU, 4 GB)
├── VM K8s-Worker-1 (8 CPU, 16 GB)
└── VM K8s-Worker-2 (8 CPU, 16 GB)
```

**Возможности:**
- До 200 одновременных студентов
- До 50 активных лабораторных работ
- High Availability (несколько worker nodes)

**Инструменты:**
- Terraform: управление всеми VMs
- Helm: все компоненты
- Python K8s Client: лабораторные работы

---

### Фаза 3: Enterprise (Multi-Cloud)

**Цель:** Масштабирование на несколько учебных заведений / регионов

**Инфраструктура:**
```
Multi-Cloud
├── Hetzner: GitLab VM
├── Proxmox (локальный): Database VM (конфиденциальные данные)
└── Yandex Cloud: Managed K8s Cluster (автоскейлинг)
```

**Возможности:**
- Неограниченное количество студентов (автоскейлинг)
- Географическое распределение
- Disaster recovery

**Инструменты:**
- **Terraform: критичен** (управление multi-cloud инфраструктурой)
- Helm: все компоненты
- Python K8s Client: лабораторные работы

---

## 10. Резюме технологий

| Компонент | Технология | Назначение |
|-----------|-----------|------------|
| **Виртуализация** | Proxmox | Гипервизор для создания VMs |
| **Provisioning** | Terraform | Создание VMs, сетей, multi-cloud |
| **Оркестрация** | Kubernetes | Управление контейнерами приложений |
| **Package Manager** | Helm | Установка статичных компонентов в K8s |
| **Динамическое управление** | Python Kubernetes Client | Создание/удаление лабораторных работ |
| **CI/CD** | GitLab CI | Автоматическая сборка и деплой |
| **Container Registry** | GitLab Container Registry | Хранение Docker образов |
| **База данных** | PostgreSQL 15 | Реляционные данные |
| **Кэш** | Redis 7 | In-memory кэш и очереди |
| **Object Storage** | Minio | S3-совместимое хранилище файлов |
| **Message Broker** | RabbitMQ 3.12 | Асинхронная коммуникация |
| **IAM** | Zitadel | Авторизация и аутентификация |
| **Backend Framework** | FastAPI (Python 3.11) | REST API сервисы |
| **Task Queue** | Celery | Асинхронные задачи |
| **Ingress** | Nginx Ingress Controller | HTTP маршрутизация в K8s |
| **Reverse Proxy** | Nginx (bare-metal) | SSL termination, внешний шлюз |

---

## 11. Ключевые решения архитектуры

### ✅ Принятые решения:

1. **Вынос данных из K8s на отдельную VM**
   - Независимость от K8s
   - Упрощение backup
   - Производительность

2. **GitLab на отдельной VM (или VPS)**
   - Экономия ресурсов K8s кластера
   - Независимость CI/CD от кластера

3. **Helm для статичных компонентов**
   - Готовые charts
   - Простота управления
   - Multi-environment через values

4. **Python Kubernetes Client для лабораторных работ**
   - Максимальная скорость
   - Полный контроль
   - Интеграция с FastAPI

5. **YAML конфигурации для преподавателей**
   - Простота (похоже на docker-compose)
   - Безопасность (валидация)
   - Гибкость

6. **Terraform для управления VMs**
   - Воспроизводимость инфраструктуры
   - Multi-environment (dev/stage/prod)
   - Готовность к multi-cloud

7. **Сетевая изоляция через Network Policies**
   - Безопасность лабораторных работ
   - Переключаемый доступ в интернет
   - Изоляция между студентами

---

## Дата последнего обновления
2025-11-29

Этот документ описывает финальную архитектуру платформы AutoLabs с учётом всех обсуждённых технических решений и компромиссов.
