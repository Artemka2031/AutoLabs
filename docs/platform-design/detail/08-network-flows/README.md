# Сетевые потоки данных (Network Flows)

## О компоненте

### Что это?

**Сетевые потоки данных (Network Flows)** — это описание всех сетевых взаимодействий между компонентами платформы AutoLabs. Документ описывает:

- **Направление трафика** между сервисами (кто к кому обращается)
- **Протоколы и порты** для каждого типа соединения
- **Последовательность вызовов** для критичных операций (sequence diagrams)
- **Сетевую безопасность** (Network Policies, Firewall правила)
- **Точки интеграции** между on-premise и облачными компонентами

Это **не программный компонент**, а архитектурная документация, помогающая понять:
- Как данные перемещаются по платформе
- Какие порты нужно открыть в firewall
- Где находятся узкие места и точки отказа
- Как настроить Network Policies в Kubernetes

### Где применяется?

Network flows охватывают **все уровни** инфраструктуры:

1. **Внешний уровень** — Пользователи → Ingress NGINX → Backend Platform
2. **Уровень приложений** — Backend Platform ↔ PostgreSQL, RabbitMQ, Zitadel
3. **Уровень оркестрации** — Celery Workers → Kubernetes API
4. **Уровень CI/CD** — GitLab → Container Registry → Kubernetes
5. **Уровень лабораторных работ** — Студенты → Lab Deployments в namespace `labs`

### Зависимости

Network flows зависят от:

**Инфраструктурные компоненты:**
- **Ingress NGINX** — точка входа для внешнего трафика
- **Kubernetes DNS (CoreDNS)** — резолвинг имён сервисов
- **Calico/CNI** — сетевая связность между pod'ами
- **Network Policies** — изоляция namespace'ов

**Сервисы приложений:**
- Backend Platform, Celery Workers, Zitadel, RabbitMQ
- PostgreSQL, Redis, Minio
- GitLab (CI/CD, Container Registry)

**Внешние зависимости (для гибридного сценария):**
- VPN или публичные endpoints для связи on-premise ↔ облако
- Managed сервисы (AWS RDS, Yandex Cloud PostgreSQL)

---

## Бизнес-логика и назначение

### Зачем документировать сетевые потоки?

1. **Безопасность:**
   - Понимание, какие сервисы должны общаться друг с другом
   - Настройка Network Policies для изоляции
   - Минимизация attack surface через принцип least privilege

2. **Отладка проблем:**
   - При сбое можно быстро определить, какое соединение нарушено
   - Понимание цепочки вызовов для трейсинга ошибок

3. **Планирование capacity:**
   - Определение узких мест (например, PostgreSQL может стать bottleneck)
   - Планирование масштабирования критичных путей

4. **Compliance и аудит:**
   - Демонстрация изоляции данных студентов
   - Проверка, что секретные данные не утекают наружу

5. **Обучение новых разработчиков:**
   - Быстрое понимание архитектуры через визуальные схемы
   - Примеры реальных запросов для разработки новых фич

### Вклад в работу платформы

Network flows — это **карта** всей платформы. Без понимания потоков данных:
- Невозможно настроить firewall и Network Policies
- Сложно отлаживать проблемы производительности
- Рискованно вносить изменения в архитектуру

---

## Основные сетевые потоки

### Схема верхнего уровня

```
┌──────────────────────────────────────────────────────────────────┐
│                         Внешний мир                               │
│  • Студенты (браузер)                                            │
│  • Преподаватели (браузер)                                       │
│  • Администраторы (браузер + SSH)                                │
└──────────────────────────────────────────────────────────────────┘
                            ↓ HTTPS (443)
┌──────────────────────────────────────────────────────────────────┐
│                    Ingress NGINX (K8s)                            │
│  • autolabs.example.com → Backend Platform                       │
│  • auth.autolabs.example.com → Zitadel                           │
└──────────────────────────────────────────────────────────────────┘
                ↓                                ↓
┌───────────────────────────┐      ┌────────────────────────────┐
│   Backend Platform (K8s)  │      │    Zitadel (K8s)           │
│   • Namespace: platform   │      │    • Namespace: auth       │
│   • Port: 8000            │      │    • Port: 8080            │
└───────────────────────────┘      └────────────────────────────┘
      ↓          ↓         ↓                    ↓
   [PostgreSQL] [RabbitMQ] [Redis]         [PostgreSQL]
      ↓
┌────────────────────────────────┐
│  Celery Workers (K8s)          │
│  • Namespace: infrastructure   │
│  • Consume from RabbitMQ       │
└────────────────────────────────┘
               ↓
         [Kubernetes API]
               ↓
┌────────────────────────────────┐
│  Lab Deployments (K8s)         │
│  • Namespace: labs             │
│  • Динамические pod'ы          │
└────────────────────────────────┘
```

---

## Детальные сетевые потоки

### 1. Аутентификация пользователя (OAuth2 Flow)

**Цель:** Пользователь входит в систему через Zitadel (OAuth2/OIDC)

```
┌─────────┐                ┌──────────────┐              ┌─────────┐              ┌────────────┐
│ Browser │                │   Backend    │              │ Zitadel │              │ PostgreSQL │
│ (User)  │                │   Platform   │              │ (OIDC)  │              │  (auth DB) │
└────┬────┘                └──────┬───────┘              └────┬────┘              └─────┬──────┘
     │                            │                           │                         │
     │ 1. GET /login              │                           │                         │
     ├───────────────────────────>│                           │                         │
     │                            │                           │                         │
     │ 2. Redirect to Zitadel     │                           │                         │
     │<───────────────────────────┤                           │                         │
     │  (OAuth2 authorize URL)    │                           │                         │
     │                            │                           │                         │
     │ 3. GET /oauth/authorize    │                           │                         │
     ├───────────────────────────────────────────────────────>│                         │
     │                            │                           │                         │
     │                            │                           │ 4. Validate credentials │
     │                            │                           ├────────────────────────>│
     │                            │                           │                         │
     │                            │                           │ 5. User data            │
     │                            │                           │<────────────────────────┤
     │                            │                           │                         │
     │ 6. Authorization code      │                           │                         │
     │<───────────────────────────────────────────────────────┤                         │
     │                            │                           │                         │
     │ 7. GET /callback?code=XXX  │                           │                         │
     ├───────────────────────────>│                           │                         │
     │                            │                           │                         │
     │                            │ 8. POST /oauth/token      │                         │
     │                            │   (exchange code for JWT) │                         │
     │                            ├──────────────────────────>│                         │
     │                            │                           │                         │
     │                            │ 9. Access Token + ID Token│                         │
     │                            │<──────────────────────────┤                         │
     │                            │                           │                         │
     │ 10. Session cookie + JWT   │                           │                         │
     │<───────────────────────────┤                           │                         │
     │                            │                           │                         │
```

**Протоколы и порты:**
- `Browser → Backend Platform`: HTTPS (443) через Ingress NGINX
- `Browser → Zitadel`: HTTPS (443) через Ingress NGINX
- `Backend Platform → Zitadel`: HTTP (8080) внутри K8s кластера
- `Zitadel → PostgreSQL`: TCP (5432) к database-vm или managed DB

**Настройки:**
- Zitadel Service: `zitadel.auth.svc.cluster.local:8080`
- PostgreSQL: `192.168.100.20:5432` (on-premise) или `<rds-endpoint>:5432` (AWS)

---

### 2. Создание лабораторной работы преподавателем

**Цель:** Teacher загружает новую лабораторную работу через Web UI

```
┌─────────┐         ┌─────────┐        ┌──────────┐        ┌─────────┐        ┌────────┐
│ Teacher │         │ Backend │        │ RabbitMQ │        │ Celery  │        │  K8s   │
│ Browser │         │Platform │        │          │        │ Worker  │        │  API   │
└────┬────┘         └────┬────┘        └────┬─────┘        └────┬────┘        └───┬────┘
     │                   │                  │                   │                  │
     │ 1. POST /api/labs │                  │                   │                  │
     │   (lab manifest)  │                  │                   │                  │
     ├──────────────────>│                  │                   │                  │
     │                   │                  │                   │                  │
     │                   │ 2. Validate RBAC │                   │                  │
     │                   │   (is Teacher?)  │                   │                  │
     │                   │                  │                   │                  │
     │                   │ 3. Save to DB    │                   │                  │
     │                   ├─────────────────────────────────────────────────────────>│
     │                   │                  │                   │                  │
     │                   │ 4. Enqueue task  │                   │                  │
     │                   │  "prepare_lab"   │                   │                  │
     │                   ├─────────────────>│                   │                  │
     │                   │                  │                   │                  │
     │ 5. 202 Accepted   │                  │                   │                  │
     │   (task_id: XXX)  │                  │                   │                  │
     │<──────────────────┤                  │                   │                  │
     │                   │                  │                   │                  │
     │                   │                  │ 6. Dequeue task   │                  │
     │                   │                  ├──────────────────>│                  │
     │                   │                  │                   │                  │
     │                   │                  │                   │ 7. Create K8s    │
     │                   │                  │                   │    resources:    │
     │                   │                  │                   │  - Namespace     │
     │                   │                  │                   │  - Deployment    │
     │                   │                  │                   │  - Service       │
     │                   │                  │                   │  - NetworkPolicy │
     │                   │                  │                   ├─────────────────>│
     │                   │                  │                   │                  │
     │                   │                  │                   │ 8. Resources OK  │
     │                   │                  │                   │<─────────────────┤
     │                   │                  │                   │                  │
     │                   │                  │ 9. Update status  │                  │
     │                   │<─────────────────────────────────────┤                  │
     │                   │   (lab ready)    │                   │                  │
     │                   │                  │                   │                  │
     │ 10. WebSocket     │                  │                   │                  │
     │     notification  │                  │                   │                  │
     │<──────────────────┤                  │                   │                  │
     │   "Lab ready!"    │                  │                   │                  │
     │                   │                  │                   │                  │
```

**Протоколы и порты:**
- `Browser → Backend Platform`: HTTPS (443) → HTTP (8000) в K8s
- `Backend Platform → PostgreSQL`: TCP (5432)
- `Backend Platform → RabbitMQ`: AMQP (5672)
- `Celery Worker → RabbitMQ`: AMQP (5672)
- `Celery Worker → Kubernetes API`: HTTPS (6443) через service account
- `Backend Platform → Browser`: WebSocket (443) для real-time уведомлений

**Важно:**
- Backend Platform **НЕ имеет** доступа к Kubernetes API (изоляция)
- Только Celery Workers взаимодействуют с K8s API
- RabbitMQ — единственная точка связи Backend ↔ Celery

---

### 3. Запуск лабораторной работы студентом

**Цель:** Student запускает свой экземпляр лабораторной работы

```
┌─────────┐      ┌─────────┐      ┌──────────┐      ┌────────┐      ┌─────────────┐
│ Student │      │ Backend │      │ RabbitMQ │      │ Celery │      │ Lab Pod     │
│ Browser │      │Platform │      │          │      │ Worker │      │ (namespace: │
│         │      │         │      │          │      │        │      │  labs)      │
└────┬────┘      └────┬────┘      └────┬─────┘      └───┬────┘      └──────┬──────┘
     │                │                 │                │                  │
     │ 1. POST        │                 │                │                  │
     │ /api/labs/123/ │                 │                │                  │
     │ start          │                 │                │                  │
     ├───────────────>│                 │                │                  │
     │                │                 │                │                  │
     │                │ 2. Check RBAC   │                │                  │
     │                │  (student in    │                │                  │
     │                │   course?)      │                │                  │
     │                │                 │                │                  │
     │                │ 3. Enqueue      │                │                  │
     │                │  "start_lab"    │                │                  │
     │                ├────────────────>│                │                  │
     │                │                 │                │                  │
     │ 4. 202 Accepted│                 │                │                  │
     │<───────────────┤                 │                │                  │
     │                │                 │                │                  │
     │                │                 │ 5. Dequeue     │                  │
     │                │                 ├───────────────>│                  │
     │                │                 │                │                  │
     │                │                 │                │ 6. Create Pod    │
     │                │                 │                │   with labels:   │
     │                │                 │                │   student=XXX    │
     │                │                 │                │   lab=123        │
     │                │                 │                ├─────────────────>│
     │                │                 │                │                  │
     │                │                 │                │ 7. Pod Running   │
     │                │                 │                │<─────────────────┤
     │                │                 │                │                  │
     │                │                 │ 8. Update DB   │                  │
     │                │<────────────────────────────────┤                  │
     │                │                 │                │                  │
     │ 9. GET         │                 │                │                  │
     │ /api/labs/123/ │                 │                │                  │
     │ connection     │                 │                │                  │
     ├───────────────>│                 │                │                  │
     │                │                 │                │                  │
     │ 10. Connection │                 │                │                  │
     │   details:     │                 │                │                  │
     │   URL, port    │                 │                │                  │
     │<───────────────┤                 │                │                  │
     │                │                 │                │                  │
     │ 11. Connect to │                 │                │                  │
     │     lab via    │                 │                │                  │
     │     Service    │                 │                │                  │
     ├────────────────────────────────────────────────────────────────────>│
     │                │                 │                │   HTTP/SSH/VNC   │
     │                │                 │                │                  │
```

**Протоколы и порты:**
- `Student → Backend Platform`: HTTPS (443)
- `Backend Platform → RabbitMQ`: AMQP (5672)
- `Celery Worker → Kubernetes API`: HTTPS (6443)
- `Student → Lab Pod`: HTTP/SSH/VNC через Ingress или NodePort
  - HTTP: 80/443 (веб-интерфейс лабы)
  - SSH: 22 (консольный доступ)
  - VNC: 5900 (графический интерфейс)

**Network Isolation:**
- Lab Pod находится в namespace `labs`
- Network Policy запрещает:
  - ❌ Lab Pod → другие Lab Pod'ы (изоляция студентов)
  - ❌ Lab Pod → platform, auth, infrastructure namespaces
  - ✅ Lab Pod → Internet (для скачивания инструментов)
  - ✅ Student (внешний) → Lab Pod (через Ingress)

---

### 4. CI/CD: Деплой новой версии Backend Platform

**Цель:** GitLab CI/CD собирает и деплоит новую версию Backend Platform

```
┌────────┐        ┌────────┐       ┌───────────┐       ┌────────┐       ┌─────────┐
│ GitLab │        │ GitLab │       │ Container │       │  Helm  │       │   K8s   │
│  Repo  │        │ Runner │       │ Registry  │       │ Chart  │       │ Cluster │
└───┬────┘        └───┬────┘       └─────┬─────┘       └───┬────┘       └────┬────┘
    │                 │                  │                 │                 │
    │ 1. git push     │                  │                 │                 │
    │    (new commit) │                  │                 │                 │
    ├────────────────>│                  │                 │                 │
    │                 │                  │                 │                 │
    │                 │ 2. Trigger       │                 │                 │
    │                 │    pipeline      │                 │                 │
    │                 │   .gitlab-ci.yml │                 │                 │
    │                 │                  │                 │                 │
    │                 │ 3. Build Docker  │                 │                 │
    │                 │    image         │                 │                 │
    │                 │                  │                 │                 │
    │                 │ 4. Scan with     │                 │                 │
    │                 │    Trivy         │                 │                 │
    │                 │   (vulnerability)│                 │                 │
    │                 │                  │                 │                 │
    │                 │ 5. Push image    │                 │                 │
    │                 ├─────────────────>│                 │                 │
    │                 │  registry.gitlab │                 │                 │
    │                 │  .com/autolabs/  │                 │                 │
    │                 │  backend:v1.2.3  │                 │                 │
    │                 │                  │                 │                 │
    │                 │ 6. helm upgrade  │                 │                 │
    │                 │   --set image.   │                 │                 │
    │                 │   tag=v1.2.3     │                 │                 │
    │                 ├─────────────────────────────────────>│                 │
    │                 │                  │                 │                 │
    │                 │                  │                 │ 7. Apply K8s    │
    │                 │                  │                 │    manifests    │
    │                 │                  │                 ├────────────────>│
    │                 │                  │                 │  - Deployment   │
    │                 │                  │                 │  - Service      │
    │                 │                  │                 │  - Ingress      │
    │                 │                  │                 │                 │
    │                 │                  │ 8. Pull image   │                 │
    │                 │                  │<────────────────────────────────────┤
    │                 │                  │                 │                 │
    │                 │                  │ 9. Image data   │                 │
    │                 │                  ├────────────────────────────────────>│
    │                 │                  │                 │                 │
    │                 │                  │                 │ 10. Rolling     │
    │                 │                  │                 │     update      │
    │                 │                  │                 │     complete    │
    │                 │                  │                 │<────────────────┤
    │                 │                  │                 │                 │
    │                 │ 11. Deployment   │                 │                 │
    │                 │     SUCCESS      │                 │                 │
    │<────────────────┤                  │                 │                 │
    │                 │                  │                 │                 │
```

**Протоколы и порты:**
- `GitLab Runner → Container Registry`: HTTPS (443)
- `GitLab Runner → Kubernetes API`: HTTPS (6443) через kubeconfig
- `Kubernetes → Container Registry`: HTTPS (443) для pull образов

**Важно:**
- GitLab Runner имеет kubeconfig с правами на deploy в namespace `platform`
- Container Registry может быть:
  - Self-hosted (registry.gitlab.com)
  - Docker Hub
  - AWS ECR / Google GCR
- ImagePullSecrets настроены в namespace для pull приватных образов

---

### 5. Мониторинг и логирование (Future)

**Цель:** Сбор метрик и логов со всех компонентов

```
┌─────────────┐       ┌────────────┐       ┌─────────┐       ┌─────────┐
│  All Pods   │       │ Prometheus │       │ Grafana │       │  Loki   │
│ (metrics,   │       │  (metrics  │       │ (visual-│       │ (logs   │
│  logs)      │       │  storage)  │       │  ization)       │ storage)│
└──────┬──────┘       └─────┬──────┘       └────┬────┘       └────┬────┘
       │                    │                   │                 │
       │ 1. Expose /metrics │                   │                 │
       │    endpoint (HTTP) │                   │                 │
       │                    │                   │                 │
       │ 2. Scrape metrics  │                   │                 │
       │<───────────────────┤                   │                 │
       │   every 15s        │                   │                 │
       │                    │                   │                 │
       │ 3. Send logs to    │                   │                 │
       │    stdout/stderr   │                   │                 │
       ├────────────────────────────────────────────────────────>│
       │                    │                   │                 │
       │                    │ 4. Query metrics  │                 │
       │                    │<──────────────────┤                 │
       │                    │                   │                 │
       │                    │ 5. Metric data    │                 │
       │                    ├──────────────────>│                 │
       │                    │                   │                 │
       │                    │                   │ 6. Query logs   │
       │                    │                   ├────────────────>│
       │                    │                   │                 │
       │                    │                   │ 7. Log data     │
       │                    │                   │<────────────────┤
       │                    │                   │                 │
```

**Протоколы и порты:**
- `Pods → Prometheus`: HTTP (9090) — Prometheus scrapes `/metrics`
- `Grafana → Prometheus`: HTTP (9090) для запросов метрик
- `Pods → Loki`: HTTP (3100) для отправки логов
- `Grafana → Loki`: HTTP (3100) для запросов логов

**Метрики (примеры):**
- `http_requests_total` — количество HTTP запросов к Backend Platform
- `celery_task_duration_seconds` — время выполнения Celery задач
- `kubernetes_pod_restarts_total` — количество перезапусков pod'ов

---

## Протоколы и порты: Справочная таблица

### Внешние порты (доступны из интернета)

| Сервис | Протокол | Порт | Назначение | Доступ |
|--------|----------|------|------------|--------|
| **Ingress NGINX** | HTTPS | 443 | Точка входа для всех HTTP(S) запросов | Все пользователи |
| **Ingress NGINX** | HTTP | 80 | Редирект на HTTPS | Все пользователи |
| **SSH (Proxmox/VM)** | SSH | 22 | Удалённое управление серверами | Только администраторы |
| **Lab SSH Access** | SSH | 2222+ | SSH доступ к лабораторным работам | Студенты (через NodePort/LB) |
| **Lab VNC Access** | VNC/HTTP | 5900+ | VNC доступ к графическим лабам | Студенты (через Ingress) |

### Внутренние порты (внутри K8s кластера)

| Сервис | Namespace | Протокол | Порт | Назначение |
|--------|-----------|----------|------|------------|
| **Backend Platform** | `platform` | HTTP | 8000 | REST API |
| **Celery Workers** | `infrastructure` | - | - | Слушают RabbitMQ |
| **Zitadel** | `auth` | HTTP | 8080 | OAuth2/OIDC сервер |
| **RabbitMQ** | `messaging` | AMQP | 5672 | Очередь задач |
| **RabbitMQ Management** | `messaging` | HTTP | 15672 | Веб-интерфейс управления |
| **PostgreSQL (autolabs)** | external | TCP | 5432 | Основная БД платформы |
| **PostgreSQL (zitadel)** | external | TCP | 5432 | БД для Zitadel |
| **Redis** | `infrastructure` | TCP | 6379 | Кэш и сессии |
| **Minio** | `infrastructure` | HTTP | 9000 | S3-compatible storage |
| **Kubernetes API** | `kube-system` | HTTPS | 6443 | K8s control plane |
| **CoreDNS** | `kube-system` | DNS | 53 | Резолвинг сервисов |

### Внешние сервисы (для on-premise)

| Сервис | VM/Host | Протокол | Порт | Назначение |
|--------|---------|----------|------|------------|
| **GitLab** | `gitlab-vm` (192.168.100.10) | HTTPS | 443 | GitLab CE веб-интерфейс |
| **GitLab SSH** | `gitlab-vm` | SSH | 22 | Git push/pull через SSH |
| **Container Registry** | `gitlab-vm` | HTTPS | 5050 | Docker Registry |
| **PostgreSQL** | `database-vm` (192.168.100.20) | TCP | 5432 | Все БД |
| **Redis** | `database-vm` | TCP | 6379 | Кэш (если не в K8s) |
| **Minio** | `database-vm` | HTTP | 9000 | S3 storage (если не в K8s) |

### Managed сервисы (для гибридного/облачного)

| Сервис | Провайдер | Протокол | Порт | Endpoint (пример) |
|--------|-----------|----------|------|-------------------|
| **AWS RDS PostgreSQL** | AWS | TCP | 5432 | `autolabs.abc123.us-east-1.rds.amazonaws.com:5432` |
| **Yandex Managed PostgreSQL** | Yandex Cloud | TCP | 6432 | `c-abc123.rw.mdb.yandexcloud.net:6432` |
| **AWS ElastiCache Redis** | AWS | TCP | 6379 | `autolabs-redis.abc123.cache.amazonaws.com:6379` |
| **AWS S3** | AWS | HTTPS | 443 | `https://autolabs-bucket.s3.amazonaws.com` |
| **Yandex Object Storage** | Yandex Cloud | HTTPS | 443 | `https://storage.yandexcloud.net/autolabs-bucket` |

---

## Network Policies в Kubernetes

**Цель:** Изоляция namespace'ов и ограничение трафика по принципу least privilege.

### Общая стратегия

```yaml
# Default Deny All (применяется ко всем namespace'ам)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <каждый namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

После этого явно разрешаем только необходимые соединения.

---

### 1. Network Policy: Backend Platform

**Namespace:** `platform`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-platform-policy
  namespace: platform
spec:
  podSelector:
    matchLabels:
      app: backend-platform
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # Разрешить трафик от Ingress NGINX
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8000

  egress:
  # Разрешить DNS запросы
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # Разрешить подключение к PostgreSQL
  - to:
    - podSelector: {}  # External IP (database-vm)
    ports:
    - protocol: TCP
      port: 5432

  # Разрешить подключение к RabbitMQ
  - to:
    - namespaceSelector:
        matchLabels:
          name: messaging
      podSelector:
        matchLabels:
          app: rabbitmq
    ports:
    - protocol: TCP
      port: 5672

  # Разрешить подключение к Redis
  - to:
    - namespaceSelector:
        matchLabels:
          name: infrastructure
      podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379

  # Разрешить подключение к Zitadel
  - to:
    - namespaceSelector:
        matchLabels:
          name: auth
      podSelector:
        matchLabels:
          app: zitadel
    ports:
    - protocol: TCP
      port: 8080
```

---

### 2. Network Policy: Celery Workers

**Namespace:** `infrastructure`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: celery-workers-policy
  namespace: infrastructure
spec:
  podSelector:
    matchLabels:
      app: celery-worker
  policyTypes:
  - Egress

  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # RabbitMQ (consume tasks)
  - to:
    - namespaceSelector:
        matchLabels:
          name: messaging
      podSelector:
        matchLabels:
          app: rabbitmq
    ports:
    - protocol: TCP
      port: 5672

  # Kubernetes API (создание lab deployments)
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 6443

  # PostgreSQL (обновление статусов)
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 5432
```

**ВАЖНО:** Celery Workers имеют доступ к Kubernetes API через ServiceAccount с RBAC правами.

---

### 3. Network Policy: Lab Deployments (Студенческая изоляция)

**Namespace:** `labs`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lab-isolation-policy
  namespace: labs
spec:
  podSelector: {}  # Применяется ко всем pod'ам в namespace labs
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # Разрешить трафик только от Ingress NGINX
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx

  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # Интернет (для скачивания инструментов в лабораторных работах)
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443

  # ЗАПРЕТ: lab pod'ы НЕ могут общаться друг с другом
  # ЗАПРЕТ: lab pod'ы НЕ могут обращаться к другим namespace'ам
```

**Результат:**
- ✅ Студент может подключиться к своей лабе через Ingress
- ✅ Лаба может скачивать инструменты из интернета
- ❌ Лаба студента A не может достучаться до лабы студента B
- ❌ Лаба не может достучаться до Backend Platform, Zitadel, RabbitMQ

---

### 4. Network Policy: Zitadel

**Namespace:** `auth`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: zitadel-policy
  namespace: auth
spec:
  podSelector:
    matchLabels:
      app: zitadel
  policyTypes:
  - Ingress
  - Egress

  ingress:
  # От Ingress NGINX (пользователи)
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080

  # От Backend Platform (token validation)
  - from:
    - namespaceSelector:
        matchLabels:
          name: platform
      podSelector:
        matchLabels:
          app: backend-platform
    ports:
    - protocol: TCP
      port: 8080

  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # PostgreSQL (zitadel DB)
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 5432
```

---

## Firewall правила (для on-premise)

### На уровне Proxmox host

```bash
# Входящий трафик
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH для администраторов
iptables -A INPUT -p tcp --dport 8006 -j ACCEPT  # Proxmox Web UI
iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # HTTPS для приложения
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # HTTP (редирект на HTTPS)

# Запретить всё остальное
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

### На уровне VM (gitlab-vm, database-vm, k8s-vm)

#### GitLab VM (192.168.100.10)

```bash
# ufw (Uncomplicated Firewall)
ufw allow 22/tcp     # SSH
ufw allow 80/tcp     # GitLab HTTP
ufw allow 443/tcp    # GitLab HTTPS
ufw allow 5050/tcp   # Container Registry

# Разрешить из K8s подсети
ufw allow from 192.168.100.30 to any port 5050  # K8s pull образов

ufw enable
```

#### Database VM (192.168.100.20)

```bash
# PostgreSQL доступен только из K8s VM и GitLab VM
ufw allow from 192.168.100.30 to any port 5432  # K8s
ufw allow from 192.168.100.10 to any port 5432  # GitLab

# Redis доступен только из K8s VM
ufw allow from 192.168.100.30 to any port 6379

# Minio доступен только из K8s VM
ufw allow from 192.168.100.30 to any port 9000

# SSH для администраторов
ufw allow 22/tcp

ufw enable
```

#### Kubernetes VM (192.168.100.30)

```bash
# SSH
ufw allow 22/tcp

# Kubernetes API (для kubectl, GitLab CI/CD)
ufw allow 6443/tcp

# NodePort range (для Lab deployments с внешним доступом)
ufw allow 30000:32767/tcp

# Ingress NGINX (если используется NodePort вместо LoadBalancer)
ufw allow 80/tcp
ufw allow 443/tcp

ufw enable
```

---

## Гибридный сценарий: On-premise ↔ Облако

### Вариант A: Публичные endpoints (простой, но менее безопасный)

```
┌─────────────────────────────────┐
│  On-premise (Proxmox)           │
│  • K8s cluster                  │
│  • Backend Platform             │
│  • Celery Workers               │
└───────────┬─────────────────────┘
            ↓ HTTPS/TCP
            ↓ Public IP
┌───────────┴─────────────────────┐
│  Облако (AWS / Yandex Cloud)    │
│  • Managed PostgreSQL           │
│    Endpoint: <public-endpoint>  │
│    Port: 5432                   │
│  • Security Group:              │
│    Allow from <on-premise IP>   │
└─────────────────────────────────┘
```

**Настройка:**
1. Managed PostgreSQL имеет публичный IP
2. Security Group разрешает подключения только от IP адреса on-premise K8s VM
3. Обязательно использовать SSL/TLS для подключения к PostgreSQL

**Преимущества:**
- ✅ Простая настройка
- ✅ Не требует VPN

**Недостатки:**
- ⚠️ База данных доступна из интернета (хоть и защищена Security Group)
- ⚠️ Трафик идёт через интернет (можно использовать SSL)

---

### Вариант B: VPN туннель (безопасный)

```
┌─────────────────────────────────┐
│  On-premise (Proxmox)           │
│  • K8s cluster                  │
│  • VPN Client (WireGuard)       │
│    10.0.0.1                     │
└───────────┬─────────────────────┘
            ↓ VPN Tunnel
            ↓ Encrypted
┌───────────┴─────────────────────┐
│  Облако (AWS / Yandex Cloud)    │
│  • VPN Server (WireGuard)       │
│    10.0.0.2                     │
│  • Private Subnet               │
│    • Managed PostgreSQL         │
│      10.0.1.10:5432             │
│    • Managed Redis              │
│      10.0.1.20:6379             │
└─────────────────────────────────┘
```

**Настройка (WireGuard):**

**On-premise (K8s VM):**
```bash
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <on-premise-private-key>
Address = 10.0.0.1/24

[Peer]
PublicKey = <cloud-public-key>
Endpoint = <cloud-vpn-server-ip>:51820
AllowedIPs = 10.0.1.0/24  # Облачная подсеть
PersistentKeepalive = 25
```

**Облако (VPN Server EC2/VM):**
```bash
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <cloud-private-key>
Address = 10.0.0.2/24
ListenPort = 51820

[Peer]
PublicKey = <on-premise-public-key>
AllowedIPs = 10.0.0.1/32
```

**Результат:**
- K8s подключается к PostgreSQL через VPN: `10.0.1.10:5432`
- PostgreSQL **не имеет публичного IP**
- Весь трафик зашифрован

**Преимущества:**
- ✅ Максимальная безопасность
- ✅ БД не доступна из интернета

**Недостатки:**
- ⚠️ Сложнее настройка
- ⚠️ VPN server нужно поддерживать

---

### Вариант C: Cloud VPN Gateway (для production)

Использовать managed VPN сервисы:
- **AWS VPN Gateway** (Site-to-Site VPN)
- **Google Cloud VPN**
- **Yandex Cloud IPSec VPN**

**Преимущества:**
- ✅ Managed сервис (не нужен VPN server)
- ✅ Высокая доступность
- ✅ Автоматический failover

**Стоимость:** ~$30-50/мес (плата за VPN Gateway)

---

## Диаграмма: Полный network flow для типичной операции

### Сценарий: Student запускает лабу и подключается к ней

```
Student Browser
      │
      │ 1. HTTPS GET /api/labs/123/start
      ▼
Ingress NGINX (443)
      │
      │ 2. HTTP (8000) to Backend Platform
      ▼
Backend Platform Pod (namespace: platform)
      │
      │ 3. TCP (5432) SELECT * FROM labs WHERE id=123
      ▼
PostgreSQL (database-vm or AWS RDS)
      │
      │ 4. Check RBAC: student assigned to lab?
      ▼
Backend Platform Pod
      │
      │ 5. AMQP (5672) enqueue task "start_lab"
      ▼
RabbitMQ Pod (namespace: messaging)
      │
      │ 6. AMQP (5672) dequeue task
      ▼
Celery Worker Pod (namespace: infrastructure)
      │
      │ 7. HTTPS (6443) POST /api/v1/namespaces/labs/pods
      ▼
Kubernetes API Server
      │
      │ 8. Create Pod in namespace: labs
      ▼
Lab Pod (namespace: labs)
      │ Pod is running, Service created
      │
      │ 9. TCP (5432) UPDATE labs SET status='running'
      ▼
PostgreSQL
      │
      │ 10. WebSocket notification to Browser
      ▼
Student Browser
      │
      │ 11. HTTPS GET /api/labs/123/connection
      ▼
Backend Platform Pod
      │
      │ 12. Returns: https://lab123.autolabs.com
      ▼
Student Browser
      │
      │ 13. HTTPS (443) to lab123.autolabs.com
      ▼
Ingress NGINX
      │
      │ 14. HTTP (80/8080) to Lab Pod
      ▼
Lab Pod (namespace: labs)
      │
      │ Student работает с лабораторной работой
      ▼
```

**Общее количество сетевых переходов:** 14 шагов
**Протоколы:** HTTPS, HTTP, TCP, AMQP, WebSocket
**Компоненты:** 7 (Browser, Ingress, Backend, PostgreSQL, RabbitMQ, Celery, K8s API, Lab Pod)

---

## Оптимизация и troubleshooting

### Узкие места (bottlenecks)

1. **PostgreSQL:**
   - Все компоненты обращаются к одной БД
   - **Решение:** Connection pooling (PgBouncer), read replicas для масштабирования

2. **RabbitMQ:**
   - Единая точка для всех Celery задач
   - **Решение:** RabbitMQ кластер (3 узла) или миграция на Kafka

3. **Ingress NGINX:**
   - Все внешние запросы проходят через Ingress
   - **Решение:** Horizontal Pod Autoscaler для Ingress NGINX

### Типичные проблемы и диагностика

**Проблема:** Student не может подключиться к лабе

**Диагностика:**
```bash
# 1. Проверить, что pod лабы запущен
kubectl get pods -n labs -l student=student123

# 2. Проверить Service
kubectl get svc -n labs -l student=student123

# 3. Проверить Ingress
kubectl get ingress -n labs

# 4. Проверить Network Policy
kubectl get networkpolicy -n labs

# 5. Проверить логи Ingress NGINX
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# 6. Curl изнутри кластера
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://lab-service.labs.svc.cluster.local:8080
```

**Проблема:** Backend Platform не может подключиться к PostgreSQL

**Диагностика:**
```bash
# 1. Проверить доступность PostgreSQL с pod'а
kubectl exec -it -n platform deployment/backend-platform -- \
  psql -h 192.168.100.20 -U autolabs_user -d autolabs

# 2. Проверить Network Policy
kubectl get networkpolicy -n platform

# 3. Проверить firewall на database-vm
ufw status

# 4. Проверить логи PostgreSQL
tail -f /var/log/postgresql/postgresql-15-main.log
```

---

## Вывод

Сетевые потоки данных AutoLabs построены на принципах:

1. **Изоляция по умолчанию:**
   - Default Deny All Network Policies
   - Явное разрешение только необходимых соединений
   - Изоляция студенческих лабораторных работ

2. **Least Privilege:**
   - Backend Platform **НЕ имеет** доступа к Kubernetes API
   - Только Celery Workers управляют инфраструктурой
   - Firewall rules разрешают минимально необходимый трафик

3. **Безопасность:**
   - HTTPS для всех внешних соединений
   - Network Policies изолируют namespace'ы
   - VPN для связи on-premise ↔ облако (для гибридного сценария)

4. **Наблюдаемость:**
   - Все компоненты экспортируют метрики (Prometheus)
   - Централизованное логирование (Loki/ELK)
   - Трейсинг для отладки сложных операций

5. **Гибкость:**
   - Поддержка on-premise, гибридного и облачного сценариев
   - Managed сервисы подключаются через VPN или публичные endpoints

**Критически важно:**

- Network Policies должны быть настроены **до** запуска production
- Firewall rules на VM обязательны для on-premise сценария
- Для гибридного сценария рекомендуется VPN (а не публичные endpoints)
- Регулярный аудит Network Policies для выявления избыточных разрешений

**Рекомендация:** Начать с default-deny-all, затем постепенно добавлять разрешения по мере тестирования компонентов.

---

## Дополнительные ресурсы

**Документация:**
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Network Policy](https://docs.tigera.io/calico/latest/network-policy/)
- [Ingress NGINX](https://kubernetes.github.io/ingress-nginx/)
- [WireGuard VPN](https://www.wireguard.com/)

**Инструменты:**
- [kubectl-network-policy](https://github.com/networkpolicy/kubectl-np) — визуализация Network Policies
- [netpol.io](https://editor.cilium.io/) — редактор Network Policies
- [Cilium Hubble](https://github.com/cilium/hubble) — наблюдение за сетевым трафиком в K8s

**Следующие этапы документации:**
- `02-backend-platform/` — Backend Platform (FastAPI, бизнес-логика)
- `03-celery-workers/` — Celery Workers (K8s API, инфраструктурные операции)
- `05-kubernetes/` — Kubernetes (namespaces, deployments, services)
- `07-lab-deployments/` — Деплой лабораторных работ студентов
