# Celery Workers — Документация компонента

## Содержание
- [О компоненте](#о-компоненте)
- [Бизнес-логика и назначение](#бизнес-логика-и-назначение)
- [Архитектурные принципы и обоснования](#архитектурные-принципы-и-обоснования)
- [Типы задач и варианты реализации](#типы-задач-и-варианты-реализации)
- [Обработка ошибок и retry механизмы](#обработка-ошибок-и-retry-механизмы)
- [Безопасность и контроль доступа](#безопасность-и-контроль-доступа)
- [Мониторинг и observability](#мониторинг-и-observability)
- [Конфигурация и развёртывание](#конфигурация-и-развёртывание)
- [Интеграция с платформой](#интеграция-с-платформой)
- [Заключение](#заключение)

---

## О компоненте

**Celery Workers** — это специализированный инфраструктурный компонент платформы AutoLabs, выполняющий асинхронные операции с Kubernetes API. Это **единственный** компонент системы, имеющий прямой доступ к K8s API для создания, модификации и удаления ресурсов лабораторных работ студентов.

### Что это?

Celery Workers — распределённая система обработки асинхронных задач, построенная на базе фреймворка Celery и использующая RabbitMQ как message broker. Workers получают задачи от Backend Platform и выполняют инфраструктурные операции, которые требуют значительного времени выполнения и не должны блокировать основной API.

### Где развёртывается?

- **Kubernetes Deployment** в namespace `platform`
- **Replicas:** 3-5 (в зависимости от нагрузки)
- **Autoscaling:** HPA на базе длины очереди RabbitMQ

### Ресурсы

| Параметр | Request | Limit | Обоснование |
|----------|---------|-------|-------------|
| **CPU** | 250m | 500m | K8s API вызовы не CPU-intensive |
| **RAM** | 256Mi | 512Mi | Минимальное состояние, работа с K8s клиентом |
| **Storage** | - | - | Stateless компонент |

### Зависимости

**Обязательные зависимости:**
- **RabbitMQ** — источник задач (message broker)
- **PostgreSQL** — обновление статусов LabInstance
- **Kubernetes API** — целевая система для выполнения операций
- **Minio** — хранилище конфигурационных файлов (kubeconfig, манифесты)

**Опциональные зависимости:**
- **Redis** — результаты выполнения задач (Celery backend)
- **Authorization Service** — дополнительная проверка прав доступа (defence in depth)

### Критически важное архитектурное решение

**Backend Platform НЕ имеет доступа к Kubernetes API.**

Это фундаментальное архитектурное ограничение, обеспечивающее:
- **Разделение ответственности** (Separation of Concerns)
- **Изоляцию бизнес-логики** от инфраструктурных операций
- **Безопасность** через минимизацию привилегированных компонентов
- **Отказоустойчивость** через асинхронную обработку

```
Backend Platform (бизнес-логика)
        ↓ публикация задачи
    RabbitMQ (очередь)
        ↓ получение задачи
Celery Workers (инфраструктура)
        ↓ K8s API вызовы
    Kubernetes (выполнение)
```

---

## Бизнес-логика и назначение

### Зачем нужны Celery Workers?

Celery Workers решают фундаментальную проблему: **изоляцию бизнес-логики от инфраструктурных операций**.

#### Проблема: Прямой доступ Backend Platform к K8s API

Если бы Backend Platform имел прямой доступ к Kubernetes API, возникли бы следующие проблемы:

1. **Смешивание ответственности**
   - Бизнес-логика (управление курсами, авторизация) смешана с инфраструктурой (создание Pod'ов)
   - Нарушение принципа Single Responsibility

2. **Блокирующие операции**
   - Создание Pod'а может занимать 10-30 секунд (pull образа, запуск контейнера)
   - HTTP запрос пользователя висит в ожидании
   - Плохой UX, риск timeout'ов

3. **Избыточные привилегии**
   - Backend Platform (публичный API) имеет доступ к K8s API
   - Большая поверхность атаки
   - Риск escalation при компрометации

4. **Сложность масштабирования**
   - Backend Platform должен масштабироваться под нагрузку API
   - Workers должны масштабироваться под нагрузку создания Pod'ов
   - Разные паттерны масштабирования конфликтуют

#### Решение: Асинхронная обработка через Celery Workers

**Celery Workers обеспечивают:**

1. **Асинхронность**
   - Backend Platform мгновенно возвращает ответ пользователю (status: pending)
   - Worker обрабатывает задачу в фоне
   - Пользователь опрашивает статус через polling или получает уведомление

2. **Изоляция привилегий**
   - Только Workers имеют ServiceAccount с правами на K8s API
   - Backend Platform работает без привилегий
   - Минимизация blast radius при компрометации

3. **Надёжность**
   - Задачи персистятся в RabbitMQ
   - При падении Worker'а задача не теряется
   - Retry механизм при временных сбоях

4. **Независимое масштабирование**
   - Backend Platform масштабируется под HTTP нагрузку
   - Workers масштабируются под нагрузку создания Pod'ов
   - HPA на базе метрик очереди

### Типичный сценарий работы

**Студент запускает лабораторную работу:**

1. **Frontend → Backend Platform**
   - `POST /api/v1/lab-instances {lab_id: 123}`
   - Backend проверяет права доступа (ABAC)
   - Backend создаёт запись `LabInstance` в PostgreSQL (status: PENDING)

2. **Backend Platform → RabbitMQ**
   - Публикация задачи `create_lab_pod`
   - Payload: `{lab_instance_id, lab_id, student_id, docker_image, resources}`
   - Task ID возвращается пользователю

3. **RabbitMQ → Celery Worker**
   - Worker получает задачу из очереди
   - Worker загружает конфигурацию из Minio
   - Worker валидирует параметры

4. **Celery Worker → Kubernetes API**
   - Создание Namespace (если не существует)
   - Создание Pod (с заданными resources)
   - Создание Service (для доступа к Pod'у)
   - Создание NetworkPolicy (изоляция)

5. **Celery Worker → PostgreSQL**
   - Обновление `LabInstance.status = RUNNING`
   - Сохранение `k8s_pod_name`, `connection_url`
   - Логирование событий

6. **Студент получает доступ**
   - Frontend опрашивает `GET /api/v1/lab-instances/{id}`
   - Backend возвращает `connection_url`
   - Студент переходит по ссылке и работает с лабой

---

## Архитектурные принципы и обоснования

### 1. Единственный компонент с доступом к K8s API

**Принцип:** Только Celery Workers имеют ServiceAccount с правами на Kubernetes API.

**Обоснование:**

- **Минимизация привилегий** (Principle of Least Privilege)
  - Backend Platform (публичный API) работает без привилегий
  - Компрометация Backend Platform не даёт доступа к K8s

- **Упрощение аудита**
  - Все K8s операции проходят через Workers
  - Единая точка логирования и мониторинга

- **Гибкость архитектуры**
  - Можно заменить Celery на другую систему очередей
  - Backend Platform не зависит от реализации инфраструктуры

### 2. Асинхронная обработка через message broker

**Принцип:** Взаимодействие Backend Platform ↔ Workers только через RabbitMQ.

**Обоснование:**

- **Надёжность**
  - Задачи персистятся в RabbitMQ (durable queues)
  - При падении Worker'а задача не теряется

- **Масштабируемость**
  - Добавление Workers не требует изменений в Backend Platform
  - Horizontal scaling на основе длины очереди

- **Отказоустойчивость**
  - RabbitMQ HA cluster с зеркалированием очередей
  - Автоматический failover при падении узла

### 3. Stateless Workers

**Принцип:** Workers не хранят состояние между задачами.

**Обоснование:**

- **Упрощение развёртывания**
  - Можно добавлять/удалять Workers без миграции данных
  - Rolling updates без downtime

- **Предсказуемость**
  - Каждая задача обрабатывается независимо
  - Нет "грязного" состояния от предыдущих задач

- **Debugging**
  - Легко воспроизводить проблемы
  - Логи задачи содержат всю нужную информацию

### 4. Defence in Depth для безопасности

**Принцип:** Многоуровневая защита доступа к K8s API.

**Уровни защиты:**

1. **Архитектурная изоляция**
   - Backend Platform физически не может обращаться к K8s API
   - Отсутствие kubeconfig и ServiceAccount

2. **RBAC на уровне Kubernetes**
   - ServiceAccount Workers имеет минимальные права
   - Только namespace `labs`, только Pod/Service/NetworkPolicy

3. **Authorization Service**
   - Дополнительная проверка прав перед выполнением задачи
   - Валидация: может ли student_id запустить lab_id

4. **Network Policy**
   - Workers могут обращаться только к K8s API и PostgreSQL
   - Изоляция от других компонентов

**Обоснование:**

- **Redundancy защиты** — компрометация одного уровня не даёт полного доступа
- **Audit trail** — логирование на каждом уровне
- **Compliance** — соответствие security best practices

### 5. Конфигурация как код в Minio

**Принцип:** Конфигурационные файлы (kubeconfig, манифесты) хранятся в Minio, а не в контейнере Workers.

**Обоснование:**

- **Безопасность**
  - Kubeconfig не лежит в Docker образе
  - Ротация credentials без пересборки образа

- **Гибкость**
  - Изменение конфигурации без пересборки образа
  - Разные конфигурации для разных окружений (dev, prod)

- **Централизованное управление**
  - Единое место хранения конфигураций
  - Версионирование через Minio versioning

---

## Типы задач и варианты реализации

### Архитектура задач

Каждая задача имеет:
- **Тип** (create_lab_pod, delete_lab_pod, cleanup_timeout)
- **Приоритет** (high, normal, low)
- **Payload** (параметры выполнения)
- **Retry policy** (количество попыток, backoff)

### Основные типы задач

#### 1. `create_lab_pod` — Создание лабораторной работы

**Назначение:** Создание всех необходимых K8s ресурсов для запуска лабораторной работы студента.

**Входные данные:**
```
{
  "lab_instance_id": 123,
  "lab_id": 456,
  "student_id": 789,
  "docker_image": "registry.gitlab.com/autolabs/labs/nmap:v1.0.0",
  "resources": {
    "cpu": "500m",
    "memory": "512Mi",
    "storage": "1Gi"
  },
  "network_isolated": true
}
```

**Варианты реализации:**

**Вариант A: Последовательное создание ресурсов**
1. Создание Pod с labels `lab_instance_id`, `student_id`
2. Ожидание Pod Ready (timeout 60 секунд)
3. Создание Service (type: ClusterIP)
4. Создание NetworkPolicy (если network_isolated=true)
5. Обновление LabInstance в PostgreSQL

**Плюсы:** Простота, чёткий порядок выполнения
**Минусы:** Долгое выполнение (блокирующее ожидание)

**Вариант B: Декларативное создание через манифесты**
1. Генерация YAML манифеста с Pod + Service + NetworkPolicy
2. Применение `kubectl apply -f manifest.yaml`
3. Асинхронное ожидание через watch механизм
4. Callback при достижении состояния Ready

**Плюсы:** Быстрее, K8s сам координирует создание
**Минусы:** Сложнее обработка ошибок

**Вариант C: Helm chart на каждую лабу**
1. Подготовка Helm chart с параметрами
2. `helm install lab-instance-123 ./lab-chart`
3. Helm управляет жизненным циклом

**Плюсы:** Стандартизация, rollback из коробки
**Минусы:** Overhead, сложность для простых лаб

**Рекомендация:** **Вариант B** для MVP, переход на Вариант C для production.

---

#### 2. `delete_lab_pod` — Удаление лабораторной работы

**Назначение:** Полное удаление ресурсов при остановке лабы студентом или по timeout.

**Входные данные:**
```
{
  "lab_instance_id": 123,
  "k8s_pod_name": "lab-instance-123-nmap",
  "force": false  # true для force delete при cleanup
}
```

**Варианты реализации:**

**Вариант A: Каскадное удаление по label selector**
```
kubectl delete all -l lab_instance_id=123 --namespace=labs
```

**Плюсы:** Одна команда удаляет всё (Pod, Service, NetworkPolicy)
**Минусы:** Потенциально опасно при ошибке в labels

**Вариант B: Последовательное удаление**
1. Удаление NetworkPolicy
2. Удаление Service
3. Удаление Pod
4. Обновление LabInstance.status = STOPPED

**Плюсы:** Контроль на каждом шаге
**Минусы:** Медленнее

**Рекомендация:** **Вариант A** с validation labels перед выполнением.

---

#### 3. `cleanup_timeout_labs` — Периодическая очистка

**Назначение:** Удаление лаб, превысивших timeout (например, 2 часа работы).

**Запуск:** Celery Beat (периодическая задача каждые 5 минут)

**Логика:**
1. Запрос к PostgreSQL: `SELECT * FROM lab_instances WHERE status='RUNNING' AND created_at < NOW() - INTERVAL '2 hours'`
2. Для каждой записи: постановка задачи `delete_lab_pod` с `force=true`

**Варианты реализации:**

**Вариант A: Celery Beat в отдельном контейнере**
- Отдельный Deployment для scheduler
- Публикация задач cleanup в очередь

**Вариант B: CronJob в Kubernetes**
- K8s CronJob запускает контейнер каждые 5 минут
- Контейнер выполняет скрипт очистки

**Рекомендация:** **Вариант A** для консистентности с остальными задачами.

---

#### 4. `scale_lab_resources` — Изменение ресурсов (будущее)

**Назначение:** Динамическое изменение CPU/RAM лабы без пересоздания Pod'а.

**Входные данные:**
```
{
  "lab_instance_id": 123,
  "new_resources": {
    "cpu": "1000m",
    "memory": "1Gi"
  }
}
```

**Реализация:**
- Patching Pod spec через K8s API
- Требует In-Place Pod Vertical Scaling (K8s 1.27+)

**Статус:** Отложено до production фазы.

---

### Приоритеты задач

**High Priority:**
- `delete_lab_pod` — освобождение ресурсов критично
- `health_check_pods` — мониторинг здоровья

**Normal Priority:**
- `create_lab_pod` — стандартные операции

**Low Priority:**
- `cleanup_timeout_labs` — фоновая очистка
- `sync_lab_status` — синхронизация статусов

**Обоснование приоритетов:**
- Удаление важнее создания (освобождаем ресурсы для новых лаб)
- Health checks критичны для observability
- Cleanup может подождать

---

## Обработка ошибок и retry механизмы

### Философия обработки ошибок

**Принцип:** Различать временные (transient) и постоянные (permanent) ошибки.

**Временные ошибки:**
- K8s API недоступен (503 Service Unavailable)
- Недостаточно ресурсов в кластере (Pod pending)
- Network timeout

**Постоянные ошибки:**
- Невалидный docker image (ImagePullBackOff)
- Недостаточные RBAC права
- Некорректные параметры задачи

**Стратегия:**
- **Временные ошибки → Retry** с exponential backoff
- **Постоянные ошибки → Immediate fail** + уведомление

### Retry конфигурация (переменные окружения)

```bash
# Базовые настройки retry
CELERY_TASK_MAX_RETRIES=3          # Максимум 3 попытки
CELERY_TASK_RETRY_DELAY=5          # Начальная задержка 5 секунд
CELERY_TASK_RETRY_BACKOFF=2        # Exponential backoff x2
CELERY_TASK_RETRY_JITTER=true      # Случайный jitter для избежания thundering herd

# Пример: 1-я попытка → fail → wait 5s → 2-я попытка → fail → wait 10s → 3-я попытка
```

**Обоснование значений:**

- **3 попытки** — баланс между надёжностью и скоростью фейла
- **Exponential backoff** — даём K8s время восстановиться при временных проблемах
- **Jitter** — избегаем одновременного retry всех задач (thundering herd)

### Система контроля и мониторинга ошибок

#### Базовый контроль (MVP)

**1. Логирование всех ошибок**
```
[ERROR] Task create_lab_pod [task_id=abc123] failed after 3 retries
Reason: ImagePullBackOff (image not found)
Lab Instance ID: 123
Student ID: 789
```

**2. Обновление статуса в PostgreSQL**
- `LabInstance.status = FAILED`
- `LabInstance.error_message = "Image not found"`
- `LabInstance.failed_at = NOW()`

**3. Dead Letter Queue (DLQ)**
- Задачи, провалившиеся после всех retry, попадают в отдельную очередь
- Ручной анализ админом

#### Расширенный контроль (Production)

**1. Категоризация ошибок**
```
ERROR_CATEGORIES = {
    "ImagePullBackOff": "PERMANENT_IMAGE_ERROR",
    "InsufficientResources": "TEMPORARY_RESOURCE_ERROR",
    "NetworkTimeout": "TEMPORARY_NETWORK_ERROR",
    "RBACDenied": "PERMANENT_PERMISSION_ERROR"
}
```

**2. Автоматические действия**
- `PERMANENT_IMAGE_ERROR` → уведомление teacher'у
- `TEMPORARY_RESOURCE_ERROR` → retry с увеличенным timeout
- `PERMANENT_PERMISSION_ERROR` → алерт админу

**3. Metrics и alerting**
- Prometheus counter: `celery_task_failures_total{error_category="..."}`
- Alert: если >10 permanent errors за 5 минут

**Обоснование подхода:**

Начинаем с простого логирования + DLQ, добавляем сложность по мере роста платформы. Преждевременная оптимизация усложняет код без явной выгоды на MVP стадии.

---

## Безопасность и контроль доступа

### Многоуровневая защита (Defence in Depth)

#### Уровень 1: Архитектурная изоляция

**Backend Platform не может создавать Pod'ы по архитектуре.**

- Отсутствие kubeconfig в контейнере Backend Platform
- Нет ServiceAccount с правами на K8s API
- Network Policy запрещает трафик от Backend Platform к K8s API

**Обоснование:**
Даже при компрометации Backend Platform (SQL injection, RCE) атакующий не получит доступ к K8s.

#### Уровень 2: RBAC на уровне Kubernetes

**ServiceAccount для Celery Workers:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: celery-workers-sa
  namespace: platform

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: celery-workers-role
  namespace: labs
rules:
# Управление Pod'ами студентов
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "list", "delete", "watch"]

# Управление Service'ами
- apiGroups: [""]
  resources: ["services"]
  verbs: ["create", "get", "list", "delete"]

# Управление NetworkPolicy
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["create", "get", "list", "delete"]

# ЗАПРЕЩЕНО:
# - Создание privileged Pod'ов
# - Доступ к Secret'ам вне namespace labs
# - Изменение RBAC
```

**Ключевые ограничения:**

- **Только namespace `labs`** — Workers не могут трогать `platform`, `infrastructure`, `auth`
- **Только Pod/Service/NetworkPolicy** — нет доступа к Deployment, StatefulSet, DaemonSet
- **Нет прав на Secret/ConfigMap** — минимизация утечки credentials

**Обоснование:**
Минимальные права для выполнения задач. Компрометация Workers не даёт полного контроля над кластером.

#### Уровень 3: Authorization Service (опционально)

**Дополнительная проверка перед выполнением задачи:**

```python
# Псевдокод в Worker
async def create_lab_pod(lab_instance_id, student_id, lab_id):
    # Дополнительная проверка через Authorization Service
    is_authorized = await auth_service.check_permission(
        student_id=student_id,
        action="start_lab",
        resource_id=lab_id
    )

    if not is_authorized:
        raise PermissionDenied("Student not authorized to start this lab")

    # Основная логика создания Pod'а
    ...
```

**Обоснование:**
RBAC K8s проверяет, может ли ServiceAccount создать Pod. Authorization Service проверяет, может ли студент запустить лабу. Два независимых слоя защиты.

#### Уровень 4: Network Policy изоляция Workers

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: celery-workers-netpol
  namespace: platform
spec:
  podSelector:
    matchLabels:
      app: celery-workers
  policyTypes:
  - Egress
  egress:
  # К K8s API
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 6443

  # К PostgreSQL
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432

  # К RabbitMQ
  - to:
    - namespaceSelector:
        matchLabels:
          name: messaging
    ports:
    - protocol: TCP
      port: 5672

  # ЗАПРЕЩЕНО: исходящий трафик к другим компонентам
```

**Обоснование:**
Ограничение lateral movement при компрометации. Workers не могут обращаться к Redis, Zitadel, GitLab.

### Защита от escalation

**Проблема:** Worker может создать privileged Pod, который захватит node.

**Решение: Pod Security Standards**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: labs
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Ограничения для Pod'ов в namespace `labs`:**
- ❌ Нет privileged режима
- ❌ Нет hostPath volumes
- ❌ Нет hostNetwork/hostPID
- ❌ Нет capabilities (кроме базовых)

**Обоснование:**
Даже если Worker скомпрометирован и пытается создать вредоносный Pod, K8s отклонит манифест.

---

## Мониторинг и observability

### Философия мониторинга

**Мониторинг должен отвечать на три вопроса:**
1. **Работают ли Workers?** (Health)
2. **Успешно ли они выполняют задачи?** (Success rate)
3. **Почему конкретная лаба не запустилась?** (Debugging)

### Ключевые метрики (общие концепции)

#### 1. Метрики очереди
- `rabbitmq_queue_length{queue="celery.lab_operations"}` — длина очереди
- `rabbitmq_queue_consumers{queue="celery.lab_operations"}` — количество Workers

**Алерт:** Если длина очереди >100 и растёт → нужно масштабировать Workers.

#### 2. Метрики задач
- `celery_tasks_started_total{task_type="create_lab_pod"}` — запущенные задачи
- `celery_tasks_succeeded_total{task_type="create_lab_pod"}` — успешные
- `celery_tasks_failed_total{task_type="create_lab_pod", error_category="..."}` — проваленные

**Алерт:** Если failure rate >5% за 5 минут → что-то сломалось (K8s API? Образы?).

#### 3. Метрики производительности
- `celery_task_duration_seconds{task_type="create_lab_pod"}` — histogram времени выполнения

**Алерт:** Если p99 latency >60 секунд → K8s медленно создаёт Pod'ы (проблема с ресурсами?).

#### 4. Метрики K8s ресурсов
- `kube_pod_status_phase{namespace="labs", phase="Running"}` — количество запущенных Pod'ов студентов
- `kube_pod_status_phase{namespace="labs", phase="Pending"}` — застрявшие Pod'ы

**Алерт:** Если >10 Pod'ов в Pending >5 минут → недостаток ресурсов в кластере.

### Логирование (концепции)

**Критичные события для логирования:**

**1. Начало задачи**
```
[INFO] Task create_lab_pod started [task_id=abc123, lab_instance_id=456, student_id=789]
```

**2. Успешное выполнение**
```
[INFO] Task create_lab_pod succeeded [task_id=abc123, pod_name=lab-instance-456, duration=12.3s]
```

**3. Ошибка**
```
[ERROR] Task create_lab_pod failed [task_id=abc123, attempt=2/3, error=ImagePullBackOff]
Reason: Image "registry.gitlab.com/autolabs/labs/nmap:v999" not found
```

**4. K8s API события**
```
[DEBUG] K8s API call: POST /api/v1/namespaces/labs/pods [response=201, duration=0.5s]
[DEBUG] K8s API call: GET /api/v1/namespaces/labs/pods/lab-instance-456 [response=200, pod_status=Running]
```

**Обоснование структуры:**

- **Structured logging** — поля можно парсить (task_id, lab_instance_id)
- **Correlation ID** — можно проследить задачу от Backend Platform до K8s
- **Уровни логирования** — INFO для бизнес-событий, DEBUG для технических деталей

### Alerting (концепции)

**Критические алерты:**

1. **Workers unavailable**
   - Условие: 0 Workers подключены к RabbitMQ >1 минуту
   - Действие: Немедленное уведомление админа

2. **High failure rate**
   - Условие: >10% задач fail за 5 минут
   - Действие: Уведомление дежурного инженера

3. **Queue overflow**
   - Условие: Длина очереди >500
   - Действие: Автоматическое масштабирование Workers (HPA)

4. **Stuck labs**
   - Условие: >10 LabInstance в PENDING >10 минут
   - Действие: Ручной анализ (проблема с образами? K8s?)

### Debugging workflow

**Проблема:** Студент жалуется "Моя лаба не запускается".

**Шаги диагностики:**

1. **Проверка LabInstance в PostgreSQL**
   ```sql
   SELECT status, error_message, celery_task_id, created_at
   FROM lab_instances
   WHERE id = 123;
   ```

2. **Проверка логов Worker'а по task_id**
   ```bash
   kubectl logs -n platform -l app=celery-workers | grep "task_id=abc123"
   ```

3. **Проверка Pod'а в K8s**
   ```bash
   kubectl get pods -n labs -l lab_instance_id=123
   kubectl describe pod lab-instance-123 -n labs
   ```

4. **Анализ событий K8s**
   ```bash
   kubectl get events -n labs --field-selector involvedObject.name=lab-instance-123
   ```

**Обоснование подхода:**
Логи должны содержать достаточно context'а (task_id, lab_instance_id, student_id), чтобы можно было проследить цепочку от HTTP запроса до Pod'а в K8s.

---

## Конфигурация и развёртывание

### Управление конфигурацией через Minio

**Проблема:** Kubeconfig и манифесты нельзя хардкодить в Docker образе.

**Решение:** Хранение конфигурации в Minio, загрузка при старте Worker'а.

**Структура в Minio:**

```
autolabs-configs/
├── kubeconfig/
│   ├── prod-cluster.yaml      # Kubeconfig для production K8s
│   └── dev-cluster.yaml       # Kubeconfig для dev K8s
├── manifests/
│   ├── lab-pod-template.yaml  # Шаблон Pod'а для лабы
│   └── network-policy-template.yaml
└── settings/
    └── celery-workers-config.yaml  # Параметры Workers
```

**Процесс загрузки конфигурации:**

1. Worker стартует, подключается к Minio
2. Загружает `kubeconfig/prod-cluster.yaml` в `/tmp/kubeconfig`
3. Устанавливает `KUBECONFIG=/tmp/kubeconfig`
4. Загружает шаблоны манифестов в память
5. Начинает обработку задач

**Обоснование:**

- **Безопасность** — kubeconfig не в образе, ротация без пересборки
- **Гибкость** — разные конфигурации для dev/staging/prod
- **Audit** — Minio логирует все обращения к kubeconfig

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: celery-workers
  namespace: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: celery-workers
  template:
    metadata:
      labels:
        app: celery-workers
    spec:
      serviceAccountName: celery-workers-sa  # RBAC права

      initContainers:
      - name: load-config
        image: minio/mc:latest
        command:
        - sh
        - -c
        - |
          mc alias set minio $MINIO_URL $MINIO_ACCESS_KEY $MINIO_SECRET_KEY
          mc cp minio/autolabs-configs/kubeconfig/prod-cluster.yaml /config/kubeconfig
        volumeMounts:
        - name: config
          mountPath: /config

      containers:
      - name: celery-worker
        image: registry.gitlab.com/autolabs/celery-workers:latest
        env:
        - name: CELERY_BROKER_URL
          value: "amqp://rabbitmq.messaging:5672"
        - name: KUBECONFIG
          value: "/config/kubeconfig"
        - name: CELERY_TASK_MAX_RETRIES
          value: "3"
        volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi

      volumes:
      - name: config
        emptyDir: {}
```

**Ключевые моменты:**

- **initContainer** загружает kubeconfig из Minio при старте Pod'а
- **serviceAccountName** привязывает RBAC права
- **emptyDir volume** — временное хранилище для kubeconfig (удаляется при рестарте)

### Horizontal Pod Autoscaling (HPA)

**Масштабирование на основе длины очереди RabbitMQ:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: celery-workers-hpa
  namespace: platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: celery-workers
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: rabbitmq_queue_length
        selector:
          matchLabels:
            queue: celery.lab_operations
      target:
        type: AverageValue
        averageValue: "20"  # Масштабируем, если >20 задач на Worker
```

**Обоснование:**

- **3 минимальных реплики** — отказоустойчивость
- **10 максимальных** — защита от runaway scaling
- **20 задач на Worker** — баланс между latency и эффективностью

---

## Интеграция с платформой

### Взаимодействие с Backend Platform

**Backend Platform → Celery Workers:**

```python
# Backend Platform: публикация задачи
from celery import Celery

celery_app = Celery(broker='amqp://rabbitmq:5672')

task_result = celery_app.send_task(
    'celery_workers.create_lab_pod',
    kwargs={
        'lab_instance_id': 123,
        'lab_id': 456,
        'student_id': 789,
        'docker_image': 'registry.gitlab.com/autolabs/labs/nmap:v1.0.0',
        'resources': {'cpu': '500m', 'memory': '512Mi'}
    }
)

# task_result.id → сохраняем в LabInstance.celery_task_id
```

**Celery Workers → Backend Platform:**

Workers НЕ вызывают API Backend Platform. Обновление статуса через прямую запись в PostgreSQL:

```python
# Celery Worker: обновление статуса
async def update_lab_instance_status(lab_instance_id, status, pod_name=None):
    await db.execute(
        "UPDATE lab_instances SET status = :status, k8s_pod_name = :pod_name WHERE id = :id",
        {"status": status, "pod_name": pod_name, "id": lab_instance_id}
    )
```

**Обоснование:**
Workers — низкоуровневый компонент, не должен зависеть от HTTP API Backend Platform. Прямой доступ к PostgreSQL проще и быстрее.

### Взаимодействие с RabbitMQ

**Queues:**

- `celery.lab_operations` — основная очередь для create/delete задач
- `celery.cleanup` — очередь для фоновой очистки (low priority)
- `celery.dlq` — Dead Letter Queue для проваленных задач

**Exchange:**

- `autolabs.tasks` (type: topic)
  - Routing key: `lab.create` → `celery.lab_operations`
  - Routing key: `lab.delete` → `celery.lab_operations`
  - Routing key: `lab.cleanup` → `celery.cleanup`

**Обоснование:**
Topic exchange даёт гибкость маршрутизации. Можно добавить отдельные Workers для разных типов задач.

### Взаимодействие с Kubernetes API

**Клиент:** Python библиотека `kubernetes-client`

```python
from kubernetes import client, config

# Загрузка конфигурации из KUBECONFIG
config.load_kube_config()

# Создание API клиентов
v1 = client.CoreV1Api()  # Для Pod, Service
networking_v1 = client.NetworkingV1Api()  # Для NetworkPolicy

# Пример: создание Pod'а
pod_manifest = {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {"name": "lab-instance-123", "namespace": "labs"},
    "spec": {...}
}

v1.create_namespaced_pod(namespace="labs", body=pod_manifest)
```

**Обоснование:**
Официальная библиотека, поддержка всех K8s API, type hints для Python.

---

## Заключение

### Ключевые архитектурные принципы

Celery Workers реализуют критическую часть платформы AutoLabs на основе следующих принципов:

1. **Единственный компонент с доступом к K8s API**
   - Изоляция привилегий
   - Минимизация blast radius
   - Упрощение аудита

2. **Асинхронная обработка через message broker**
   - Надёжность (персистентность задач)
   - Масштабируемость (независимое scaling)
   - Отказоустойчивость (failover)

3. **Defence in Depth для безопасности**
   - Архитектурная изоляция
   - RBAC на уровне K8s
   - Authorization Service
   - Network Policy

4. **Stateless дизайн**
   - Упрощение deployment
   - Предсказуемость
   - Лёгкое масштабирование

5. **Конфигурация как код в Minio**
   - Безопасность (ротация credentials)
   - Гибкость (разные окружения)
   - Audit trail

### Критически важные аспекты

**Backend Platform НЕ имеет доступа к Kubernetes API.**
Это не просто техническое решение — это фундаментальный архитектурный принцип, обеспечивающий:
- Разделение бизнес-логики и инфраструктуры
- Минимизацию поверхности атаки
- Независимую эволюцию компонентов

**Retry и обработка ошибок настраиваются через окружение.**
Начинаем с простых значений (3 retry, 5s delay), оптимизируем на основе реальных данных production среды.

**Мониторинг — обязателен, но начинаем с базового.**
Логирование + метрики очереди + failure rate достаточно для MVP. Добавляем сложность по мере роста.

### Рекомендации для дальнейшей работы

1. **MVP фаза:**
   - Реализовать `create_lab_pod` и `delete_lab_pod` (Вариант B — декларативные манифесты)
   - Базовое логирование + Dead Letter Queue
   - RBAC с минимальными правами
   - HPA на основе длины очереди

2. **Production фаза:**
   - Добавить `cleanup_timeout_labs` через Celery Beat
   - Категоризация ошибок + автоматические действия
   - Prometheus metrics + Grafana dashboards
   - Authorization Service для дополнительной проверки

3. **Optimization фаза:**
   - Helm charts для лаб (если нужна стандартизация)
   - In-place Pod scaling (если нужно динамическое изменение ресурсов)
   - Distributed tracing (OpenTelemetry)

### Дополнительные ресурсы

- [Celery Documentation](https://docs.celeryproject.org/)
- [Kubernetes Python Client](https://github.com/kubernetes-client/python)
- [RabbitMQ Best Practices](https://www.rabbitmq.com/best-practices.html)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

---

**Следующий этап документации:**
- `05-kubernetes/` — Kubernetes (namespaces, RBAC, network policies)
- `07-lab-deployments/` — Lab Deployments (студенческие Pod'ы, изоляция)
