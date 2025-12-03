# Backend Platform (Бэкенд платформы)

## О компоненте

### Что это?

**Backend Platform** — это ядро бизнес-логики платформы AutoLabs, реализованное на **FastAPI** с применением принципов **Domain-Driven Design (DDD)** и **Clean Architecture**.

Backend Platform отвечает за:
- **REST API** для фронтенда и внешних интеграций
- **Бизнес-логику** управления курсами, лабораторными работами, пользователями
- **Авторизацию и контроль доступа** через Zitadel (OAuth2/OIDC)
- **Оркестрацию задач** через RabbitMQ для Celery Workers
- **Управление данными** через PostgreSQL

**ВАЖНО:** Backend Platform **НЕ имеет прямого доступа к Kubernetes API**. Все операции с K8s (создание pod'ов, services, deployments) выполняются через **Celery Workers**, которые получают задачи из RabbitMQ.

### Где разворачивается?

- **Тип:** Kubernetes Deployment
- **Namespace:** `platform`
- **Replicas:** 3 (для отказоустойчивости)

### Ресурсы

| Параметр | Request | Limit |
|----------|---------|-------|
| **CPU** | 500m (0.5 core) | 1000m (1 core) |
| **RAM** | 512Mi | 1Gi |
| **Storage** | - | - (stateless) |
| **Порт** | 8000 (HTTP) | - |

### Зависимости

**Backend Platform зависит от:**

1. **PostgreSQL** (192.168.100.20:5432 или managed) — основная БД
   - База данных: `autolabs`
   - Хранит: пользователей, курсы, лабы, экземпляры лаб, заявки на доступ

2. **Redis** (infrastructure namespace:6379) — кеширование
   - Кеш пользователей, курсов, разрешений
   - Сессии пользователей (TTL: 1 час)

3. **RabbitMQ** (messaging namespace:5672) — очередь задач
   - Отправка задач Celery Workers для операций с K8s
   - Получение результатов выполнения задач

4. **Zitadel** (auth namespace:8080) — авторизация
   - OAuth2/OIDC аутентификация
   - Управление пользователями и группами
   - Валидация JWT токенов

5. **Ingress NGINX** — точка входа для HTTP запросов
   - `autolabs.example.com` → Backend Platform

**Backend Platform НЕ зависит от:**
- ❌ Kubernetes API (изоляция!)
- ❌ Celery Workers (асинхронное взаимодействие через RabbitMQ)

---

## Бизнес-логика и назначение

### Зачем нужен Backend Platform?

Backend Platform — это **мозг** всей системы AutoLabs. Он обеспечивает:

1. **Централизованную бизнес-логику:**
   - Управление курсами (создание, назначение студентов)
   - Управление лабами (создание, одобрение, публикация)
   - Запуск экземпляров лаб для студентов
   - Контроль доступа (RBAC + ABAC)

2. **Единую точку входа для API:**
   - REST API для веб-интерфейса
   - Документация через OpenAPI (Swagger)
   - Версионирование API (`/api/v1/`)

3. **Оркестрацию между сервисами:**
   - Координация между PostgreSQL, Redis, RabbitMQ, Zitadel
   - Асинхронное взаимодействие с Celery Workers
   - Кеширование для оптимизации производительности

4. **Безопасность:**
   - Аутентификация через Zitadel (OAuth2/OIDC)
   - Авторизация по ролям (Admin, Teacher, Student)
   - Attribute-based access control (владелец курса, назначен на курс)

### Вклад в работу платформы

```
┌─────────────┐
│   Browser   │ (Пользователи)
└──────┬──────┘
       │ HTTPS
       ↓
┌─────────────┐
│Ingress NGINX│
└──────┬──────┘
       │ HTTP
       ↓
┌──────────────────────────────────────┐
│     Backend Platform (FastAPI)       │
│  • REST API (курсы, лабы, юзеры)     │
│  • Бизнес-логика                     │
│  • RBAC/ABAC контроль доступа        │
└──┬─────┬──────┬───────┬──────────────┘
   │     │      │       │
   │     │      │       ↓
   │     │      │    [Zitadel]
   │     │      │    OAuth2/OIDC
   │     │      │
   │     │      ↓
   │     │   [Redis]
   │     │   Кеширование
   │     │
   │     ↓
   │  [RabbitMQ]
   │  Задачи для Celery Workers
   │
   ↓
[PostgreSQL]
БД платформы
```

**Типичный сценарий:**
1. Студент запрашивает запуск лабы через веб-интерфейс
2. Backend Platform проверяет права доступа (ABAC)
3. Backend Platform создаёт запись `LabInstance` в PostgreSQL (status=pending)
4. Backend Platform отправляет задачу в RabbitMQ: `"start_lab"`
5. Celery Worker получает задачу и создаёт Pod в K8s
6. Celery Worker обновляет `LabInstance` (status=running)
7. Backend Platform возвращает студенту connection URL

---

## Архитектура и принципы

### Domain-Driven Design (DDD) + Clean Architecture

Backend Platform следует **4-слойной архитектуре**:

```
┌─────────────────────────────────────────┐
│         API Layer (Presentation)        │  ← FastAPI routes, Pydantic schemas
├─────────────────────────────────────────┤
│      Application Layer (Use Cases)      │  ← Controllers, DTOs, Orchestration
├─────────────────────────────────────────┤
│         Domain Layer (Business)         │  ← Entities, Services, Business Logic
├─────────────────────────────────────────┤
│    Infrastructure Layer (Technical)     │  ← Repositories, External APIs, Cache
└─────────────────────────────────────────┘
```

**Принцип инверсии зависимостей (Dependency Inversion):**
- Зависимости направлены **внутрь**, к Domain Layer
- API Layer → Application Layer → Domain Layer
- Infrastructure Layer реализует интерфейсы из Domain Layer
- Domain Layer **НЕ зависит** от Infrastructure

### Структура проекта

```
backend-platform/
├── src/
│   ├── main.py                    # Точка входа FastAPI
│   ├── api/                       # API Layer
│   │   └── v1/
│   │       ├── auth/              # /api/v1/auth
│   │       ├── courses/           # /api/v1/courses
│   │       ├── labs/              # /api/v1/labs
│   │       ├── lab_instances/     # /api/v1/lab-instances
│   │       ├── users/             # /api/v1/users
│   │       └── access_requests/   # /api/v1/access-requests
│   ├── application/               # Application Layer
│   │   └── controllers/
│   │       ├── course_controller.py
│   │       ├── lab_controller.py
│   │       ├── lab_instance_controller.py
│   │       └── dto.py
│   ├── domain/                    # Domain Layer
│   │   ├── auth/                  # Домен авторизации
│   │   │   ├── auth_service.py
│   │   │   └── decorators.py
│   │   ├── courses/               # Домен курсов
│   │   │   └── course_service.py
│   │   ├── labs/                  # Домен лабораторных работ
│   │   │   └── lab_service.py
│   │   ├── lab_instances/         # Домен экземпляров лаб
│   │   │   └── lab_instance_service.py
│   │   ├── access/                # Домен контроля доступа
│   │   │   ├── access_manager.py
│   │   │   └── permission_checker.py
│   │   └── tasks/                 # Домен задач (RabbitMQ)
│   │       └── task_service.py
│   ├── infrastructure/            # Infrastructure Layer
│   │   ├── repositories/
│   │   │   ├── course_repository.py
│   │   │   ├── lab_repository.py
│   │   │   └── user_repository.py
│   │   ├── auth/
│   │   │   ├── zitadel_client.py
│   │   │   └── token_manager.py
│   │   ├── cache/
│   │   │   └── cache_manager.py
│   │   ├── messaging/
│   │   │   └── rabbitmq_client.py
│   │   └── middleware/
│   ├── core/                      # DI контейнер и интерфейсы
│   │   ├── container.py
│   │   └── interfaces/
│   ├── database/
│   │   ├── models.py              # SQLAlchemy модели
│   │   ├── base.py                # BaseRepository
│   │   ├── session.py
│   │   └── migrations/            # Alembic
│   ├── config/
│   │   └── settings.py            # Pydantic Settings
│   └── utils/
├── tests/
│   ├── unit/
│   └── integration/
├── pyproject.toml                 # Poetry
├── Dockerfile
└── alembic.ini
```

### Основные домены (Domain Layer)

Backend Platform разделён на **изолированные домены**, каждый со своей ответственностью:

#### 1. **Auth Domain** (Домен авторизации)

**Ответственность:**
- Аутентификация пользователей через Zitadel (OAuth2/OIDC)
- Валидация JWT токенов
- Управление сессиями
- Разрешение ролей из Zitadel groups

**Ключевые компоненты:**
- `AuthService` — аутентификация и валидация токенов
- `TokenManager` — извлечение и валидация JWT
- `SessionService` — управление сессиями в Redis

**Роли пользователей:**
- **Admin** — полный доступ ко всем ресурсам
- **Teacher** — создание курсов, управление своими курсами
- **Student** — доступ только к назначенным курсам и лабам

#### 2. **Courses Domain** (Домен курсов)

**Ответственность:**
- Создание и управление курсами
- Назначение студентов на курсы
- Привязка лабораторных работ к курсам
- Проверка прав доступа (ABAC: owner_of_course, assigned_to_course)

**Ключевые сущности:**
- `Course` — курс (название, описание, владелец)
- `CourseStudent` — связь курс ↔ студенты (many-to-many)
- `CourseLab` — связь курс ↔ лабораторные работы

#### 3. **Labs Domain** (Домен лабораторных работ)

**Ответственность:**
- Создание лабораторных работ
- Управление статусами (draft → pending_approval → approved)
- Workflow одобрения лаб (Teacher создаёт → Admin одобряет)
- Хранение метаданных (docker image, resources, network_isolated)

**Ключевые сущности:**
- `Lab` — лабораторная работа
  - `docker_image` — образ для деплоя
  - `resources` — CPU, RAM, storage
  - `status` — draft, pending_approval, approved, rejected

#### 4. **LabInstances Domain** (Домен экземпляров лаб)

**Ответственность:**
- Запуск экземпляров лаб для студентов
- Отправка задач в RabbitMQ для Celery Workers
- Отслеживание статусов (pending → running → stopped)
- Предоставление connection details (URL, port)

**Ключевые сущности:**
- `LabInstance` — экземпляр лабораторной работы
  - `lab_id`, `student_id`, `course_id`
  - `status` — pending, running, stopped, failed
  - `k8s_pod_name`, `connection_url`
  - `celery_task_id` — ID задачи в RabbitMQ

**Бизнес-правила:**
- Студент не может запустить более 1 экземпляра одной и той же лабы одновременно
- Студент может иметь несколько активных экземпляров разных лаб
- При остановке экземпляра Pod удаляется из K8s

#### 5. **Access Domain** (Домен контроля доступа)

**Ответственность:**
- RBAC (Role-Based Access Control) — проверка ролей
- ABAC (Attribute-Based Access Control) — проверка атрибутов
  - `owner_of_course` — владелец курса
  - `assigned_to_course` — назначен на курс
  - `created_lab` — создатель лабы

**Ключевые компоненты:**
- `AccessManager` — централизованный менеджер доступа
- `PermissionChecker` — проверка разрешений
- `RoleResolver` — разрешение ролей из Zitadel groups

**Примеры проверок:**
```
Teacher может редактировать курс:
  - IF user.role == "teacher" AND user.id == course.owner_id

Student может запустить лабу:
  - IF user.role == "student"
    AND course_student.student_id == user.id
    AND course_lab.course_id == course.id
    AND course_lab.lab_id == lab.id
    AND lab.status == "approved"
```

#### 6. **Tasks Domain** (Домен задач)

**Ответственность:**
- Отправка задач в RabbitMQ для Celery Workers
- Получение результатов выполнения задач
- Обновление статусов в БД на основе результатов

**Типы задач:**
- `start_lab` — запуск экземпляра лабы (создание Pod, Service, NetworkPolicy)
- `stop_lab` — остановка экземпляра лабы (удаление Pod)
- `cleanup_lab` — очистка ресурсов (после timeout)

---

## Технологический стек

### Основные библиотеки

```toml
[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.115.0"
uvicorn = {extras = ["standard"], version = "^0.32.0"}
sqlalchemy = {extras = ["asyncio"], version = "^2.0.36"}
alembic = "^1.14.0"
asyncpg = "^0.30.0"
pydantic = "^2.10.0"
pydantic-settings = "^2.6.0"
dependency-injector = "^4.42.0"
celery = "^5.4.0"
aio-pika = "^9.5.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
httpx = "^0.28.0"
redis = {extras = ["hiredis"], version = "^5.2.0"}
loguru = "^0.7.3"

[tool.poetry.group.dev.dependencies]
pytest = "^8.3.0"
pytest-asyncio = "^0.24.0"
black = "^24.10.0"
ruff = "^0.8.0"
```

### Принципы выбора технологий

1. **FastAPI** — современный async веб-фреймворк
   - Автоматическая генерация OpenAPI документации
   - Type hints и валидация через Pydantic
   - Высокая производительность (async/await)

2. **SQLAlchemy 2.0** — async ORM
   - Асинхронная работа с PostgreSQL через AsyncPG
   - Type hints поддержка
   - Миграции через Alembic

3. **Pydantic v2** — валидация данных
   - Request/Response схемы
   - Settings через `pydantic-settings`
   - Автоматическая конвертация типов

4. **dependency-injector** — DI контейнер
   - Управление зависимостями между слоями
   - Singleton, Factory, Callable провайдеры
   - Интеграция с FastAPI Depends

5. **Celery** — асинхронные задачи
   - Отправка задач в RabbitMQ
   - Получение результатов через Redis backend
   - **Backend Platform только отправляет задачи, не выполняет**

---

## API Structure (Структура API)

### Версионирование

Все эндпоинты находятся под префиксом `/api/v1/`:

```
/api/v1/auth/*            # Аутентификация
/api/v1/users/*           # Пользователи
/api/v1/courses/*         # Курсы
/api/v1/labs/*            # Лабораторные работы
/api/v1/lab-instances/*   # Экземпляры лаб
/api/v1/access-requests/* # Заявки на доступ
/api/v1/health            # Health check
```

### Основные эндпоинты

**Auth:**
```
POST   /api/v1/auth/login          # OAuth2 redirect to Zitadel
GET    /api/v1/auth/callback       # OAuth2 callback
POST   /api/v1/auth/logout
GET    /api/v1/auth/me             # Current user info
```

**Courses:**
```
GET    /api/v1/courses             # List courses (filtered by access)
POST   /api/v1/courses             # Create course (teacher)
GET    /api/v1/courses/{id}
PUT    /api/v1/courses/{id}
DELETE /api/v1/courses/{id}
POST   /api/v1/courses/{id}/students  # Assign students
```

**Labs:**
```
GET    /api/v1/labs                # List labs
POST   /api/v1/labs                # Create lab (admin)
GET    /api/v1/labs/{id}
PUT    /api/v1/labs/{id}
DELETE /api/v1/labs/{id}
POST   /api/v1/labs/{id}/approve  # Approve lab (admin)
```

**Lab Instances:**
```
GET    /api/v1/lab-instances       # My lab instances
POST   /api/v1/lab-instances       # Start lab (student)
GET    /api/v1/lab-instances/{id}
POST   /api/v1/lab-instances/{id}/stop
GET    /api/v1/lab-instances/{id}/connection  # Connection URL
```

### Пагинация и фильтрация

```
GET /api/v1/courses?offset=0&limit=50&sort=-created_at
GET /api/v1/labs?status=approved&course_id=123
GET /api/v1/lab-instances?status=running
```

### Формат ответов

**Успешный ответ (200 OK):**
```json
{
  "id": 123,
  "name": "Python Basic Lab",
  "status": "approved",
  "docker_image": "registry.gitlab.com/autolabs/labs/python-basic:v1.0.0",
  "resources": {
    "cpu": "500m",
    "memory": "512Mi",
    "storage": "1Gi"
  },
  "created_at": "2025-12-01T10:00:00Z"
}
```

**Ошибка (400/401/403/404/500):**
```json
{
  "detail": "Lab not found"
}
```

---

## Авторизация и безопасность

### OAuth2/OIDC через Zitadel

**Flow:**
1. Пользователь нажимает "Войти"
2. Backend Platform редиректит на Zitadel (`/oauth/authorize`)
3. Пользователь авторизуется в Zitadel
4. Zitadel возвращает authorization code
5. Backend Platform обменивает code на JWT access token
6. Backend Platform сохраняет session в Redis (TTL: 1 час)
7. Frontend получает session cookie

**Проверка аутентификации:**
- Декоратор `@require_authenticated_user` на каждом эндпоинте
- Извлечение JWT из cookies или Authorization header
- Валидация JWT через публичный ключ Zitadel (локально, без запросов к API)

### RBAC (Role-Based Access Control)

**Роли:**
- **Admin** — полный доступ ко всем ресурсам
- **Teacher** — создание курсов, управление своими курсами, создание заявок на лабы
- **Student** — доступ к назначенным курсам и лабам

**Проверка роли:**
```python
@require_authenticated_user
@require_role(["admin"])
async def create_lab(...):
    # Только админы могут создавать лабы
    pass
```

### ABAC (Attribute-Based Access Control)

**Атрибуты:**
- `owner_of_course` — пользователь является владельцем курса
- `assigned_to_course` — студент назначен на курс
- `created_lab` — пользователь создал лабу

**Проверка атрибутов:**
```python
# Teacher может редактировать только свои курсы
if user.role == "teacher" and course.owner_id != user.id:
    raise PermissionError("You can only edit your own courses")

# Student может запустить лабу только если назначен на курс
if user.role == "student":
    is_assigned = await check_student_assigned_to_course(user.id, course_id)
    if not is_assigned:
        raise PermissionError("You are not assigned to this course")
```

---

## Взаимодействие с внешними сервисами

### PostgreSQL (Async)

**Подключение:**
```
postgresql+asyncpg://user:pass@192.168.100.20:5432/autolabs
```

**Session management:**
- AsyncSession через `async_sessionmaker`
- Dependency Injection через FastAPI `Depends(get_async_session)`
- Connection pool: 10 connections

### Redis (Кеширование)

**Назначение:**
- Кеш пользователей (TTL: 5 минут)
- Кеш курсов (TTL: 10 минут)
- Кеш разрешений (TTL: 10 минут)
- Сессии пользователей (TTL: 1 час)

**Подключение:**
```
redis://infrastructure-redis.infrastructure.svc.cluster.local:6379/0
```

### RabbitMQ (Задачи)

**Назначение:**
- Отправка задач Celery Workers
- Получение результатов выполнения

**Подключение:**
```
amqp://user:pass@rabbitmq.messaging.svc.cluster.local:5672/
```

**Типы задач:**
```python
# Отправка задачи на запуск лабы
task_id = await rabbitmq_client.send_task(
    "celery_workers.start_lab",
    kwargs={"lab_instance_id": 123, "lab_id": 456, "student_id": 789}
)

# Результат обрабатывается Celery Worker'ом
# Backend Platform только отправляет задачи, НЕ ВЫПОЛНЯЕТ
```

### Zitadel (Авторизация)

**Подключение:**
```
https://zitadel.auth.svc.cluster.local:8080
```

**Операции:**
- OAuth2 Authorization Code Flow
- Валидация JWT токенов
- Получение информации о пользователе (`/oauth/v2/userinfo`)
- Получение групп пользователя (для разрешения ролей)

---

## Кеширование и оптимизация

### Стратегия кеширования

**Кешируемые данные:**
- **Пользователи** (TTL: 5 минут) — после первой загрузки
- **Курсы** (TTL: 10 минут) — список курсов пользователя
- **Разрешения** (TTL: 10 минут) — результаты ABAC проверок
- **Группы Zitadel** (TTL: 10 минут) — группы пользователя

**НЕ кешируемые данные:**
- ❌ Lab instances (статус меняется часто)
- ❌ Access requests (критичные данные)

### Инвалидация кеша

**Автоматическая инвалидация:**
- При создании/обновлении курса → инвалидация кеша курсов
- При назначении студента на курс → инвалидация разрешений
- При изменении лабы → инвалидация связанных курсов

**Batch операции:**
- Загрузка данных batch'ами (100 записей за раз)
- Кеширование batch результатов

---

## Deployment и DevOps

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-platform
  namespace: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-platform
  template:
    metadata:
      labels:
        app: backend-platform
    spec:
      initContainers:
      - name: run-migrations
        image: registry.gitlab.com/autolabs/backend-platform:latest
        command: ["alembic", "upgrade", "head"]
        envFrom:
        - secretRef:
            name: backend-secrets
      containers:
      - name: backend-platform
        image: registry.gitlab.com/autolabs/backend-platform:latest
        ports:
        - containerPort: 8000
        env:
        - name: DEBUG
          value: "false"
        envFrom:
        - secretRef:
            name: backend-secrets
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /api/v1/health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v1/health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Dockerfile

```dockerfile
FROM python:3.12-slim AS base

WORKDIR /app

# Install Poetry
RUN pip install --no-cache-dir poetry==1.8.0

# Copy dependencies
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false \
    && poetry install --no-dev --no-interaction

# Production stage
FROM python:3.12-slim

WORKDIR /app

COPY --from=base /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY ./src /app/src
COPY alembic.ini /app/

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/api/v1/health')"

# Run
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### CI/CD через GitLab

**Pipeline stages:**
1. **Build** — сборка Docker образа
2. **Test** — запуск unit и integration тестов
3. **Scan** — Trivy security scan
4. **Push** — push образа в Container Registry
5. **Deploy** — Helm upgrade в K8s

**GitLab Runner** имеет kubeconfig с правами на deploy в namespace `platform`.

---

## Мониторинг и логирование

### Health Check

```
GET /api/v1/health
```

**Проверяет:**
- ✅ PostgreSQL connection
- ✅ Redis connection
- ✅ RabbitMQ connection

**Ответ:**
```json
{
  "status": "healthy",
  "checks": {
    "database": "ok",
    "redis": "ok",
    "rabbitmq": "ok"
  },
  "version": "1.0.0"
}
```

### Metrics (Future)

**Prometheus metrics endpoint:**
```
GET /metrics
```

**Метрики:**
- `http_requests_total` — количество HTTP запросов
- `http_request_duration_seconds` — время обработки запросов
- `database_queries_total` — количество запросов к БД
- `cache_hits_total` / `cache_misses_total` — попадания в кеш
- `celery_tasks_sent_total` — количество отправленных задач

### Логирование

**Loguru** для структурированных логов:

```python
from loguru import logger

logger.info("User {user_id} started lab {lab_id}", user_id=123, lab_id=456)
logger.error("Failed to connect to PostgreSQL: {error}", error=str(e))
```

**Уровни логирования:**
- `DEBUG` — детальная отладочная информация (только в dev)
- `INFO` — информационные сообщения (основные операции)
- `WARNING` — предупреждения (deprecated методы, долгие запросы)
- `ERROR` — ошибки (исключения, недоступность сервисов)
- `CRITICAL` — критические ошибки (падение приложения)

**Centralized logging (Future):**
- Отправка логов в Loki через promtail
- Визуализация в Grafana

---

## Тестирование

### Структура тестов

```
tests/
├── unit/                    # Unit тесты
│   ├── test_auth_service.py
│   ├── test_course_service.py
│   └── test_lab_service.py
└── integration/             # Интеграционные тесты
    ├── test_api_courses.py
    ├── test_api_labs.py
    └── test_api_lab_instances.py
```

### Unit тесты

**Цель:** Тестирование изолированной бизнес-логики без внешних зависимостей.

**Моки:**
- PostgreSQL → Mock Repository
- Redis → Mock Cache
- RabbitMQ → Mock Task Service
- Zitadel → Mock Auth Service

### Integration тесты

**Цель:** Тестирование API эндпоинтов с реальными зависимостями (тестовая БД).

**Тестовая БД:**
- PostgreSQL в Docker (через pytest fixtures)
- Миграции применяются перед каждым тестом
- Rollback после каждого теста

**Запуск тестов:**
```bash
poetry run pytest                    # Все тесты
poetry run pytest tests/unit/        # Только unit тесты
poetry run pytest tests/integration/ # Только integration тесты
poetry run pytest --cov=src          # С покрытием кода
```

---

## Недостатки и планы улучшения

### Текущие ограничения MVP

1. **Отсутствие WebSockets / Server-Sent Events**
   - Статусы лаб обновляются через polling (GET /api/v1/lab-instances/{id})
   - **План:** Добавить WebSocket endpoint для real-time обновлений

2. **Простое кеширование**
   - Только базовое TTL кеширование
   - **План:** Добавить умную инвалидацию через события

3. **Нет multi-tenancy**
   - Все данные в одной БД без изоляции по организациям
   - **План:** Добавить `organization_id` для multi-tenancy

4. **Ограниченная observability**
   - Нет Prometheus metrics
   - Нет distributed tracing
   - **План:** Добавить OpenTelemetry для трейсинга

### Планы развития

1. **Event Bus** для междоменного взаимодействия
2. **GraphQL API** в дополнение к REST
3. **Rate Limiting** для защиты от abuse
4. **API Gateway** для централизованной аутентификации
5. **Background Jobs** для периодических задач (cleanup, notifications)

---

## Вывод

Backend Platform построен на принципах:

1. **Domain-Driven Design** — чистая бизнес-логика в Domain Layer
2. **Clean Architecture** — инверсия зависимостей, изоляция слоёв
3. **Изоляция ответственности** — Backend Platform НЕ имеет доступа к K8s API
4. **Async/await** — высокая производительность через асинхронность
5. **Type safety** — type hints везде для предотвращения ошибок
6. **Dependency Injection** — управление зависимостями через контейнер
7. **Тестируемость** — unit и integration тесты для критичной логики

**Критически важно:**

- Backend Platform является **оркестратором** бизнес-логики
- Все операции с Kubernetes выполняются через **Celery Workers**
- Асинхронное взаимодействие через **RabbitMQ**
- Чистая архитектура позволяет легко **тестировать** и **масштабировать**

**Рекомендация:** Начать с MVP (основные CRUD операции), затем добавлять сложные фичи (WebSocket, metrics, tracing) по мере роста.

---

## Дополнительные ресурсы

**Документация:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)

**Следующие этапы документации:**
- `03-celery-workers/` — Celery Workers (K8s API, инфраструктурные операции)
- `05-kubernetes/` — Kubernetes (namespaces, deployments, services)
- `07-lab-deployments/` — Деплой лабораторных работ студентов
