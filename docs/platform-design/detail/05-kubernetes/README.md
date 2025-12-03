# Kubernetes — Документация платформы

## Содержание
- [О компоненте](#о-компоненте)
- [Роль в платформе AutoLabs](#роль-в-платформе-autolabs)
- [Архитектура кластера](#архитектура-кластера)
- [Организация namespaces](#организация-namespaces)
- [RBAC и управление доступом](#rbac-и-управление-доступом)
- [Network Policies и сетевая изоляция](#network-policies-и-сетевая-изоляция)
- [Resource Management](#resource-management)
- [Storage и персистентность](#storage-и-персистентность)
- [Ключевые вопросы для проработки](#ключевые-вопросы-для-проработки)
- [Заключение](#заключение)

---

## О компоненте

**Kubernetes** — это оркестратор контейнеров и фундамент всей платформы AutoLabs. Все компоненты системы (Backend Platform, Celery Workers, PostgreSQL, Redis, RabbitMQ, Zitadel, GitLab) разворачиваются и управляются через Kubernetes.

### Что это в контексте AutoLabs?

Kubernetes для AutoLabs — это не просто платформа оркестрации, а:

1. **Единая среда выполнения** для всех компонентов платформы
2. **Система изоляции** студенческих лабораторных работ
3. **Механизм управления ресурсами** (CPU, RAM, storage)
4. **Платформа безопасности** (Network Policies, RBAC, Pod Security)
5. **Инфраструктура для масштабирования** (HPA, node autoscaling)

### Где развёртывается?

**MVP вариант:**
- **Single-node кластер** на Proxmox VM
- Control plane + Worker на одной ноде
- Kubeadm для инициализации кластера

**Production вариант:**
- **Multi-node кластер** (3 control plane nodes + N worker nodes)
- High Availability для control plane
- Managed Kubernetes (опционально: EKS, GKE, Yandex Managed K8s)

### Версия и дистрибутив

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| **Версия K8s** | 1.28+ | Поддержка Pod Security Standards, Gateway API (future) |
| **Container Runtime** | containerd | Стандарт индустрии, легковесный |
| **CNI Plugin** | Calico | Network Policies support, производительность |
| **Ingress Controller** | NGINX Ingress | Проверенное решение, широкая поддержка |

### Критически важные концепции

**Kubernetes — это не просто платформа для запуска контейнеров.**

В контексте AutoLabs, Kubernetes обеспечивает:

1. **Изоляцию студенческих лабораторных работ**
   - Каждая лаба — отдельный Pod в namespace `labs`
   - Network Policies предотвращают доступ между лабами
   - Resource Quotas ограничивают потребление ресурсов

2. **Мультитенантность на уровне namespace**
   - `platform` — Backend Platform, Celery Workers
   - `infrastructure` — PostgreSQL, Redis
   - `messaging` — RabbitMQ
   - `auth` — Zitadel
   - `labs` — студенческие Pod'ы (изолированный namespace)

3. **Декларативное управление инфраструктурой**
   - Infrastructure as Code через YAML манифесты
   - GitOps подход (будущее: ArgoCD/FluxCD)
   - Версионирование конфигураций в Git

4. **Автоматическое восстановление**
   - Self-healing при падении Pod'ов
   - Automatic rescheduling при падении node
   - Liveness/Readiness probes для контроля здоровья

---

## Роль в платформе AutoLabs

### Архитектурное видение

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌────────────────┐  │
│  │   Namespace   │  │   Namespace   │  │   Namespace    │  │
│  │   platform    │  │infrastructure │  │      labs      │  │
│  │               │  │               │  │                │  │
│  │ • Backend     │  │ • PostgreSQL  │  │ • Student Lab  │  │
│  │ • Celery      │  │ • Redis       │  │   Pod #1       │  │
│  │   Workers     │  │               │  │ • Student Lab  │  │
│  │               │  │               │  │   Pod #2       │  │
│  └───────────────┘  └───────────────┘  │ • ...          │  │
│                                        └────────────────┘  │
│  ┌───────────────┐  ┌───────────────┐                      │
│  │   Namespace   │  │   Namespace   │                      │
│  │     auth      │  │   messaging   │                      │
│  │               │  │               │                      │
│  │ • Zitadel     │  │ • RabbitMQ    │                      │
│  └───────────────┘  └───────────────┘                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Ingress NGINX (namespace: ingress-nginx)     │  │
│  │  autolabs.example.com → Backend Platform             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Разделение ответственности

**Kubernetes управляет:**
- Жизненным циклом Pod'ов (создание, рестарт, удаление)
- Networking (Service, Ingress, Network Policies)
- Storage (PersistentVolume, PersistentVolumeClaim)
- Secrets и ConfigMaps
- Resource allocation (CPU, RAM quotas)

**Celery Workers управляют:**
- Бизнес-логикой создания лабораторных работ
- Параметрами Pod'ов (docker image, resources, labels)
- Обновлением статусов в PostgreSQL

**Kubernetes НЕ знает:**
- Какой студент запустил лабу (это в PostgreSQL)
- Какие права доступа у пользователя (это в Zitadel)
- Бизнес-логику курсов и лабораторных работ (это в Backend Platform)

**Обоснование:**
Чистое разделение инфраструктуры (Kubernetes) и бизнес-логики (Backend Platform). Kubernetes — платформа, на которой всё работает, но не владеет бизнес-данными.

---

## Архитектура кластера

### MVP: Single-Node кластер

**Конфигурация:**
- **1 VM в Proxmox**
  - 8 vCPU
  - 16 GB RAM
  - 200 GB SSD
- **Control Plane + Worker** на одной ноде
- **Локальное хранилище** (local-path provisioner)

**Обоснование для MVP:**
- **Простота развёртывания** — один kubeadm init
- **Низкие требования к ресурсам** — нет overhead для HA control plane
- **Быстрый старт** — фокус на разработке, а не на инфраструктуре

**Ограничения:**
- ❌ Нет отказоустойчивости control plane
- ❌ Падение VM = падение всей платформы
- ❌ Нельзя обновлять кластер без downtime
- ❌ Ограниченная масштабируемость (только vertical scaling)

**Приемлемо для:**
- Разработки и тестирования
- Diploma project демонстрации
- Proof of Concept
- Малое количество студентов (до 20-30)

---

### Production: Multi-Node кластер

**Конфигурация:**

**Control Plane (3 ноды):**
- 3 VM в Proxmox (или managed K8s)
- По 4 vCPU, 8 GB RAM на каждую
- etcd распределённый (HA режим)

**Worker Nodes (3+ ноды):**
- По 8 vCPU, 16 GB RAM на каждую
- Автоматическое добавление нод при необходимости

**Load Balancer для Control Plane:**
- HAProxy или Keepalived для виртуального IP
- Или managed load balancer (если облако)

**Обоснование для Production:**

1. **High Availability**
   - Control plane может пережить падение 1 ноды
   - etcd quorum (2 из 3) продолжает работу
   - Pod'ы автоматически перезапускаются на здоровых нодах

2. **Масштабируемость**
   - Можно добавлять Worker nodes без остановки
   - Horizontal scaling для увеличения нагрузки

3. **Rolling Updates**
   - Обновление кластера без downtime
   - Cordon → Drain → Update → Uncordon паттерн

4. **Fault Isolation**
   - Инфраструктурные компоненты на отдельных нодах
   - Студенческие лабы на выделенных Worker nodes
   - Taint + Toleration для специализации нод

**Переход от MVP к Production:**

Это критически важный вопрос для дальнейшей проработки. Когда переходить?

---

### Managed Kubernetes (опциональный вариант)

**Провайдеры:**
- AWS EKS
- Google GKE
- Yandex Managed Kubernetes
- Azure AKS

**Преимущества:**
- ✅ Control Plane управляется провайдером (no ops)
- ✅ Автоматические обновления
- ✅ Встроенный мониторинг и логирование
- ✅ Интеграция с облачными сервисами (Load Balancers, Storage)

**Недостатки:**
- ❌ Стоимость (managed service fee)
- ❌ Зависимость от провайдера (vendor lock-in)
- ❌ Меньше контроля над конфигурацией

**Обоснование:**

Для production развёртывания в облаке managed Kubernetes — предпочтительный вариант. Экономия операционных затрат перевешивает стоимость сервиса. Для on-premise — self-managed кластер в Proxmox.

---

## Организация namespaces

### Философия разделения

**Принцип:** Каждая логическая группа компонентов — отдельный namespace.

**Обоснование:**
- **Изоляция** — Resource Quotas, Network Policies работают на уровне namespace
- **RBAC** — права доступа назначаются на namespace
- **Логическая группировка** — легко понять, где что находится
- **Blast radius** — проблема в одном namespace не влияет на другие

### Структура namespaces

#### 1. `platform` — Платформенные компоненты

**Компоненты:**
- Backend Platform (FastAPI)
- Celery Workers

**Ресурсы:**
- Deployments, Services, ConfigMaps, Secrets
- ServiceAccount для Celery Workers (с RBAC правами)

**Resource Quota:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: platform-quota
  namespace: platform
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
```

**Обоснование:**
Ограничиваем потребление ресурсов платформенными компонентами, чтобы освободить место для студенческих лаб.

---

#### 2. `infrastructure` — Базы данных и кеш

**Компоненты:**
- PostgreSQL (StatefulSet)
- Redis (Deployment)

**Особенности:**
- Использование PersistentVolumeClaims для данных
- Stricter Network Policies (доступ только от platform)
- Backup стратегия (CronJob для pg_dump)

**Resource Quota:**
```yaml
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    persistentvolumeclaims: "3"  # PostgreSQL, Redis (если нужно)
```

**Обоснование:**
Базы данных — критичные компоненты, требуют персистентного хранилища и защиты от случайного удаления.

---

#### 3. `messaging` — Message Broker

**Компоненты:**
- RabbitMQ (StatefulSet или Deployment)

**Особенности:**
- Durable queues для надёжности
- Management UI (доступен только админам)
- Мониторинг через Prometheus exporter

---

#### 4. `auth` — Аутентификация

**Компоненты:**
- Zitadel (StatefulSet)
- PostgreSQL для Zitadel (отдельный от основной БД)

**Особенности:**
- Strict Network Policy (доступ только от Ingress и Backend Platform)
- Персистентное хранилище для Zitadel БД

---

#### 5. `labs` — Студенческие лабораторные работы

**Особенность:** Это самый **критичный** namespace с точки зрения безопасности.

**Компоненты:**
- Pod'ы студентов (динамически создаются Celery Workers)
- Service для доступа к лабам
- NetworkPolicy для изоляции

**Resource Quota (пример для 50 студентов):**
```yaml
spec:
  hard:
    requests.cpu: "25"        # 50 студентов x 500m CPU
    requests.memory: 25Gi     # 50 студентов x 512Mi RAM
    limits.cpu: "50"
    limits.memory: 50Gi
    pods: "100"               # Запас для одновременного запуска нескольких лаб
```

**LimitRange (ограничение на каждый Pod):**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-pod-limits
  namespace: labs
spec:
  limits:
  - max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: "100m"
      memory: 128Mi
    default:
      cpu: "500m"
      memory: 512Mi
    defaultRequest:
      cpu: "500m"
      memory: 512Mi
    type: Container
```

**Обоснование:**

- **LimitRange** предотвращает создание "жадных" Pod'ов, которые съедят все ресурсы кластера
- **ResourceQuota** ограничивает общее потребление namespace `labs`
- Студент не может запустить Pod с 16 GB RAM (max: 1Gi)

**Pod Security Standard:**
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

**Что это даёт:**
- ❌ Нельзя создать privileged Pod
- ❌ Нельзя использовать hostPath, hostNetwork
- ❌ Нельзя запускать от root пользователя
- ❌ Нельзя добавлять capabilities

**Критически важно:**
Студенческие лабы должны быть максимально ограничены. Даже если студент найдёт способ выполнить код в Pod'е, он не сможет скомпрометировать node или другие Pod'ы.

---

#### 6. `ingress-nginx` — Ingress Controller

**Компоненты:**
- NGINX Ingress Controller (DaemonSet или Deployment)
- LoadBalancer Service (или NodePort для on-premise)

**Особенности:**
- Единая точка входа для всех HTTP/HTTPS запросов
- TLS termination
- Rate limiting (защита от DDoS)

---

#### 7. `monitoring` (опционально, будущее)

**Компоненты:**
- Prometheus
- Grafana
- Loki (логи)
- Alertmanager

**Обоснование:**
Отдельный namespace для observability компонентов. Не мешается с production компонентами.

---

### Namespace Isolation Matrix

| Namespace | Доступ к K8s API | Network Policy | RBAC |
|-----------|------------------|----------------|------|
| **platform** | ❌ (кроме Celery Workers) | Egress: PostgreSQL, Redis, RabbitMQ | ServiceAccount для Workers |
| **infrastructure** | ❌ | Ingress: только от platform | Нет |
| **messaging** | ❌ | Ingress: только от platform | Нет |
| **auth** | ❌ | Ingress: Ingress NGINX, platform | Нет |
| **labs** | ❌ | Strict isolation (Pod-to-Pod blocked) | Нет |
| **ingress-nginx** | ✅ (для маршрутизации) | Egress: все namespaces | ServiceAccount |

---

## RBAC и управление доступом

### Философия RBAC в AutoLabs

**Принцип Least Privilege:** Каждый компонент получает только те права, которые необходимы для его работы.

### ServiceAccount для Celery Workers

**Единственный компонент с правами на K8s API.**

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
# Управление Pod'ами
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "list", "delete", "watch"]

# Управление Service
- apiGroups: [""]
  resources: ["services"]
  verbs: ["create", "get", "list", "delete"]

# Управление NetworkPolicy
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["create", "get", "list", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: celery-workers-binding
  namespace: labs
subjects:
- kind: ServiceAccount
  name: celery-workers-sa
  namespace: platform
roleRef:
  kind: Role
  name: celery-workers-role
  apiGroup: rbac.authorization.k8s.io
```

**Ключевые ограничения:**

1. **Только namespace `labs`** (Role, не ClusterRole)
2. **Только Pod, Service, NetworkPolicy** (нет Secret, ConfigMap, Deployment)
3. **Нет прав на изменение RBAC** (нельзя эскалировать права)

**Обоснование:**

Даже если Celery Worker скомпрометирован, атакующий не может:
- Создать Pod в namespace `platform` или `infrastructure`
- Прочитать Secrets с credentials
- Изменить RBAC правила

---

### ServiceAccount для Ingress NGINX

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx-sa
  namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx-role
rules:
# Чтение Ingress ресурсов (для маршрутизации)
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]

# Чтение Services и Endpoints (для backend discovery)
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]

# Обновление статуса Ingress
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses/status"]
  verbs: ["update"]
```

**Обоснование:**
Ingress Controller должен видеть Ingress ресурсы во всех namespaces (ClusterRole), но не может их изменять (только watch).

---

### Человеческий доступ (kubectl)

**Роли для администраторов:**

1. **Platform Admin** — полный доступ ко всем namespaces
2. **Developer** — доступ к namespace `platform` (read-only logs, describe pods)
3. **Support** — read-only доступ для troubleshooting

**Обоснование:**

Разработчики не должны иметь доступ к namespace `infrastructure` (PostgreSQL credentials в Secrets). Принцип разделения обязанностей.

---

## Network Policies и сетевая изоляция

### Философия Network Policies

**По умолчанию запретить всё, явно разрешить необходимое.**

### Default Deny Policy для namespace `labs`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: labs
spec:
  podSelector: {}  # Применяется ко всем Pod'ам
  policyTypes:
  - Ingress
  - Egress
```

**Эффект:** Pod'ы в namespace `labs` не могут:
- Обращаться друг к другу
- Обращаться к другим namespaces
- Выходить в интернет

**Обоснование:**
Студенческие лабы должны быть изолированы. Даже если студент запустит вредоносный код, он не сможет атаковать другие Pod'ы.

---

### Разрешающие политики для `labs`

**1. Доступ к DNS:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: labs
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

**2. Доступ в интернет (опционально):**
```yaml
egress:
- to:
  - podSelector: {}
  ports:
  - protocol: TCP
    port: 80
  - protocol: TCP
    port: 443
```

**Обоснование:**
Некоторые лабы требуют скачивания инструментов из интернета (apt-get install, pip install). Это контролируемый риск.

---

### Network Policy для Backend Platform

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-platform-netpol
  namespace: platform
spec:
  podSelector:
    matchLabels:
      app: backend-platform
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # Только от Ingress NGINX
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx

  egress:
  # К PostgreSQL
  - to:
    - namespaceSelector:
        matchLabels:
          name: infrastructure
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432

  # К Redis
  - to:
    - namespaceSelector:
        matchLabels:
          name: infrastructure
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379

  # К RabbitMQ
  - to:
    - namespaceSelector:
        matchLabels:
          name: messaging
    ports:
    - protocol: TCP
      port: 5672

  # К Zitadel
  - to:
    - namespaceSelector:
        matchLabels:
          name: auth
    ports:
    - protocol: TCP
      port: 8080
```

**Обоснование:**

Backend Platform НЕ может:
- Обращаться к K8s API (нет egress правила на port 6443)
- Обращаться к namespace `labs`
- Принимать трафик от чего-то кроме Ingress

Это архитектурное ограничение, реализованное на уровне сети.

---

## Resource Management

### Философия управления ресурсами

**Проблема:** Студент может запустить Pod с 16 GB RAM и "убить" кластер.

**Решение:** Трёхуровневая защита.

### Уровень 1: LimitRange на namespace `labs`

**Ограничение на каждый Pod:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-pod-limits
  namespace: labs
spec:
  limits:
  - max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: "100m"
      memory: 128Mi
    default:
      cpu: "500m"
      memory: 512Mi
    type: Container
```

**Эффект:**
- Студент запрашивает Pod без limits → автоматически 500m CPU, 512Mi RAM
- Студент запрашивает Pod с 2 CPU → **отклонено** (max: 1 CPU)

---

### Уровень 2: ResourceQuota на namespace `labs`

**Ограничение на весь namespace:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: labs-quota
  namespace: labs
spec:
  hard:
    requests.cpu: "25"       # Всего 25 CPU для всех студентов
    requests.memory: 25Gi    # Всего 25 GB RAM
    limits.cpu: "50"
    limits.memory: 50Gi
    pods: "100"              # Максимум 100 Pod'ов одновременно
```

**Эффект:**
- 50 студентов запустили по 1 Pod'у с 500m CPU → quota исчерпана
- 51-й студент пытается запустить Pod → **отклонено**
- Backend Platform получает ошибку "quota exceeded" → информирует студента

---

### Уровень 3: Валидация в Celery Workers

**Перед созданием Pod'а проверяем:**
```python
# Проверка доступных ресурсов в namespace labs
current_usage = await k8s_client.get_resource_usage(namespace="labs")
if current_usage.cpu + requested_cpu > LABS_QUOTA_CPU:
    raise InsufficientResourcesError("Not enough CPU in cluster")
```

**Обоснование:**
Celery Worker может дать пользователю понятное сообщение "Кластер перегружен, попробуйте позже" вместо загадочной ошибки K8s.

---

### Priority Classes (будущее)

**Идея:** Критичные компоненты (Backend Platform) имеют higher priority, чем студенческие лабы.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: platform-priority
value: 1000
globalDefault: false

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: lab-priority
value: 100
globalDefault: false
```

**Эффект:**
При нехватке ресурсов K8s может evict'нуть студенческий Pod, чтобы освободить место для Backend Platform.

**Обоснование:**
Лучше временно остановить 1 студенческую лабу, чем уронить весь Backend Platform.

---

## Storage и персистентность

### Типы данных и требования к storage

**Stateful компоненты:**
- PostgreSQL (критичные данные, требуют backups)
- Redis (опционально персистентный, можно в памяти)
- Zitadel PostgreSQL (критичные данные аутентификации)
- RabbitMQ (durable queues, требуют персистентности)

**Stateless компоненты:**
- Backend Platform
- Celery Workers
- Студенческие лабы (ephemeral, удаляются после работы)

### Storage Classes

**Для on-premise (Proxmox):**

**1. local-path (default):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```

**Использование:** Общие данные (не критично при потере node)

**2. proxmox-csi (рекомендуется для production):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: proxmox-ssd
provisioner: csi.proxmox.com
parameters:
  storage: "local-lvm"
```

**Использование:** PostgreSQL, критичные данные

**Обоснование:**
local-path — простой вариант для MVP. proxmox-csi — для production с возможностью миграции PV между нодами.

---

**Для облака:**

**1. AWS EBS:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
```

**2. GCP Persistent Disk:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
```

**Обоснование:**
Managed storage в облаке: автоматические snapshots, encryption at rest, высокая доступность.

---

### PersistentVolumeClaim для PostgreSQL

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: infrastructure
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: proxmox-ssd
  resources:
    requests:
      storage: 50Gi
```

**Обоснование:**
50 GB достаточно для хранения данных курсов, лаб, пользователей на начальном этапе. Можно расширить позже через volume expansion.

---

### Backup стратегия (критичный вопрос)

**Что нужно бэкапить:**
- PostgreSQL (курсы, лабы, пользователи, lab instances)
- Zitadel PostgreSQL (пользователи, credentials)
- Конфигурации в Minio

**Методы:**

1. **CronJob для pg_dump:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: infrastructure
spec:
  schedule: "0 2 * * *"  # Каждый день в 2:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command:
            - sh
            - -c
            - pg_dump $DATABASE_URL | gzip > /backup/backup-$(date +\%Y\%m\%d).sql.gz
          restartPolicy: OnFailure
```

2. **Volume Snapshots (если CSI driver поддерживает):**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
  namespace: infrastructure
spec:
  volumeSnapshotClassName: proxmox-snapshot
  source:
    persistentVolumeClaimName: postgres-pvc
```

**Это критичный вопрос для дальнейшей проработки.**

---

## Ключевые вопросы для проработки

Эти вопросы требуют детального анализа и тестирования:

### 1. **Когда переходить от single-node к multi-node?**

**Критерии:**
- Количество студентов > 30-50 одновременно
- Потребность в HA (недопустимо падение платформы)
- Требование к SLA (99.9% uptime)

**Что нужно проработать:**
- Миграция stateful компонентов (PostgreSQL, RabbitMQ)
- Настройка HA control plane (etcd, kube-apiserver)
- Load balancer для control plane
- Стоимость дополнительных VM

---

### 2. **High Availability для control plane**

**Проблема:** Single control plane — single point of failure.

**Решение:** 3 control plane nodes + etcd quorum.

**Вопросы:**
- Как организовать load balancing для kube-apiserver?
- HAProxy или managed load balancer?
- Как обрабатывать split-brain scenarios?
- Мониторинг здоровья control plane?

**Требует тестирования:**
- Failover сценарии
- Поведение при падении etcd node

---

### 3. **Upgrade стратегия Kubernetes**

**Проблема:** Обновление K8s без downtime платформы.

**Варианты:**

**Вариант A: In-place upgrade (single-node)**
- kubeadm upgrade apply
- Downtime на время обновления (10-20 минут)

**Вариант B: Rolling upgrade (multi-node)**
- Cordon → Drain → Upgrade → Uncordon для каждой ноды
- Без downtime

**Вопросы:**
- Как часто обновлять K8s? (каждый minor release или только LTS?)
- Тестирование обновлений на dev окружении?
- Rollback план при проблемах?

---

### 4. **Backup и Disaster Recovery**

**Критичные вопросы:**

1. **Что бэкапить?**
   - PostgreSQL данные
   - etcd (состояние кластера K8s)
   - Конфигурации (YAML манифесты в Git)
   - Secrets и ConfigMaps

2. **Как часто?**
   - PostgreSQL: каждый день
   - etcd: каждый час (или continuous backup)

3. **Где хранить backups?**
   - Minio (S3-compatible)
   - Облачное хранилище (AWS S3, GCS)
   - Отдельная VM (вне K8s кластера)

4. **Recovery Time Objective (RTO)?**
   - Сколько downtime допустимо при disaster?
   - 1 час? 24 часа?

5. **Recovery Point Objective (RPO)?**
   - Сколько данных можно потерять?
   - Последний час? Последний день?

**Требует проработки:**
- Автоматизация backup процесса
- Тестирование restore процедуры (регулярно!)
- Документирование disaster recovery плана

---

### 5. **Security Hardening**

**Вопросы для проработки:**

1. **Pod Security Standards:**
   - Везде ли применены restricted политики?
   - Как обрабатывать лабы, которым нужен привилегированный доступ (например, Wireshark)?

2. **Secrets Management:**
   - Использовать встроенные K8s Secrets (base64) или external provider (HashiCorp Vault)?
   - Encryption at rest для Secrets в etcd?

3. **Image Security:**
   - Сканирование Docker образов на уязвимости (Trivy)?
   - Private registry с RBAC?
   - Подпись образов (Cosign)?

4. **Network Policies:**
   - Полное покрытие всех namespaces?
   - Тестирование политик (как убедиться, что они работают)?

5. **Audit Logging:**
   - Включить K8s audit logs?
   - Куда отправлять (Loki, Elasticsearch)?
   - Retention policy?

---

### 6. **Resource Management и автомасштабирование**

**Вопросы:**

1. **Horizontal Pod Autoscaling (HPA):**
   - Для каких компонентов использовать?
   - Backend Platform: масштабировать по CPU или по количеству HTTP запросов?
   - Celery Workers: масштабировать по длине очереди RabbitMQ?

2. **Vertical Pod Autoscaling (VPA):**
   - Автоматическая корректировка requests/limits?
   - Риски (pod restarts)?

3. **Cluster Autoscaler:**
   - Автоматическое добавление Worker nodes при нехватке ресурсов?
   - Только для облачных провайдеров или можно для Proxmox?

4. **Resource Quotas:**
   - Как балансировать между платформенными компонентами и студенческими лабами?
   - Что делать при исчерпании quota?

---

### 7. **Monitoring и Observability**

**Вопросы:**

1. **Метрики:**
   - Prometheus для сбора метрик?
   - Какие метрики критичны (node CPU, memory, disk, pod restarts)?
   - Retention period для метрик?

2. **Логирование:**
   - Centralized logging (Loki, Elasticsearch)?
   - Какие логи собирать (все pod logs, audit logs, K8s events)?
   - Retention policy?

3. **Alerting:**
   - Когда бить тревогу (node down, pod crashlooping, quota exceeded)?
   - Каналы уведомлений (email, Telegram, Slack)?

4. **Dashboards:**
   - Grafana для визуализации?
   - Готовые dashboards или custom?

**Требует проработки:**
- Выбор monitoring stack (Prometheus + Grafana или managed решение?)
- Настройка alert rules
- Интеграция с Telegram для уведомлений

---

## Заключение

### Роль Kubernetes в платформе AutoLabs

Kubernetes — это **фундамент** всей платформы, обеспечивающий:

1. **Оркестрацию** всех компонентов (Backend, Workers, PostgreSQL, etc.)
2. **Изоляцию** студенческих лабораторных работ
3. **Управление ресурсами** через Quotas и LimitRanges
4. **Безопасность** через Network Policies и RBAC
5. **Масштабируемость** через HPA и node autoscaling

### Ключевые архитектурные принципы

1. **Namespace изоляция**
   - Каждая логическая группа компонентов — отдельный namespace
   - ResourceQuota и NetworkPolicy на уровне namespace

2. **Defence in Depth для namespace `labs`**
   - Pod Security Standards (restricted)
   - Network Policies (default deny)
   - LimitRange (защита от resource exhaustion)

3. **RBAC минимальных привилегий**
   - Только Celery Workers имеют доступ к K8s API
   - Только namespace `labs`, только Pod/Service/NetworkPolicy

4. **Stateful vs Stateless**
   - Stateful компоненты (PostgreSQL) используют PersistentVolumes
   - Stateless компоненты (Backend Platform) легко масштабируются

5. **Декларативная конфигурация**
   - Infrastructure as Code (YAML манифесты)
   - Версионирование в Git
   - GitOps подход (будущее)

### MVP vs Production

**MVP (single-node):**
- ✅ Быстрый старт
- ✅ Простота управления
- ✅ Низкие требования к ресурсам
- ❌ Нет HA
- ❌ Ограниченная масштабируемость

**Production (multi-node):**
- ✅ High Availability
- ✅ Горизонтальное масштабирование
- ✅ Rolling updates без downtime
- ❌ Сложность настройки
- ❌ Больше требований к ресурсам

### Критические вопросы требующие проработки

1. **Переход от single-node к multi-node** — критерии, процесс миграции
2. **Backup и Disaster Recovery** — стратегия, автоматизация, тестирование
3. **Security Hardening** — Secrets management, image scanning, audit logs
4. **Monitoring** — выбор стека, alert rules, dashboards

### Рекомендации для следующих шагов

**Этап 1: MVP развёртывание**
- Single-node кластер в Proxmox
- Базовые Network Policies
- LimitRange и ResourceQuota для `labs`
- local-path storage

**Этап 2: Production готовность**
- Multi-node кластер (3 control + 3 workers)
- HA control plane
- Backup автоматизация
- Monitoring (Prometheus + Grafana)

**Этап 3: Optimization**
- HPA для компонентов
- Cluster Autoscaler
- GitOps (ArgoCD)
- Advanced monitoring (distributed tracing)

---

**Следующий этап документации:**
- `07-lab-deployments/` — Lab Deployments (детали создания Pod'ов для студентов, шаблоны, безопасность)

---

## Дополнительные ресурсы

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Network Policies Best Practices](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
