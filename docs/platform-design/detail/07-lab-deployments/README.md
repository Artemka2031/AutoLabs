# Lab Deployments — Документация компонента

## Содержание
- [О компоненте](#о-компоненте)
- [Роль в платформе AutoLabs](#роль-в-платформе-autolabs)
- [Жизненный цикл лабораторной работы](#жизненный-цикл-лабораторной-работы)
- [Архитектура Pod'а для лабы](#архитектура-podа-для-лабы)
- [Типы лабораторных работ](#типы-лабораторных-работ)
- [Networking и доступ студента](#networking-и-доступ-студента)
- [Безопасность и изоляция](#безопасность-и-изоляция)
- [Resource Management](#resource-management)
- [Cleanup и автоматическое удаление](#cleanup-и-автоматическое-удаление)
- [Ключевые вопросы для проработки](#ключевые-вопросы-для-проработки)
- [Заключение](#заключение)

---

## О компоненте

**Lab Deployments** — это не отдельный компонент в классическом смысле, а **архитектурный паттерн** для создания и управления студенческими лабораторными работами в Kubernetes. Каждая лабораторная работа, запущенная студентом, представляет собой изолированный Pod в namespace `labs`.

### Что это?

Lab Deployment — это процесс создания полностью изолированного окружения для выполнения студентом практического задания по кибербезопасности. Каждый студент получает:

- **Собственный Pod** с предустановленными инструментами (nmap, wireshark, metasploit, и т.д.)
- **Изолированную сеть** (не может видеть другие лабы)
- **Ограниченные ресурсы** (CPU, RAM, storage)
- **Временное окружение** (удаляется после timeout или вручную)
- **URL для доступа** (через браузер или SSH)

### Критически важная концепция

**Каждая лаба — это эфемерный (ephemeral) Pod.**

**Обоснование:**

1. **Изоляция** — студент не может повлиять на лабы других студентов
2. **Чистое состояние** — каждый запуск лабы начинается с одинакового состояния
3. **Безопасность** — после завершения работы все данные удаляются
4. **Экономия ресурсов** — неактивные лабы автоматически удаляются

**Альтернатива (не используется):**

Persistent Pod для каждого студента на весь семестр. Проблемы:
- ❌ Высокое потребление ресурсов (Pod висит весь семестр, даже когда студент не работает)
- ❌ "Грязное" состояние (студент может сломать окружение и не сможет продолжить)
- ❌ Сложность backup'а состояния

---

## Роль в платформе AutoLabs

### Архитектурное видение

Lab Deployments — это **конечная цель** всей платформы. Вся инфраструктура (Backend Platform, Celery Workers, Kubernetes, Network Policies) существует для одной цели: **дать студенту безопасное изолированное окружение для практики по кибербезопасности.**

```
┌──────────────────────────────────────────────────────────────┐
│                    Kubernetes Namespace: labs                │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │  Lab Pod #1   │  │  Lab Pod #2   │  │  Lab Pod #3     │  │
│  │  Student: 123 │  │  Student: 456 │  │  Student: 789   │  │
│  │  Lab: nmap    │  │  Lab: metaspl │  │  Lab: wireshark │  │
│  │               │  │               │  │                 │  │
│  │ • Container   │  │ • Container   │  │ • Container     │  │
│  │   - nmap      │  │   - metasploit│  │   - wireshark   │  │
│  │   - netcat    │  │   - postgres  │  │   - tcpdump     │  │
│  │               │  │               │  │                 │  │
│  │ • Service     │  │ • Service     │  │ • Service       │  │
│  │   ClusterIP   │  │   ClusterIP   │  │   ClusterIP     │  │
│  │               │  │               │  │                 │  │
│  └───────────────┘  └───────────────┘  └─────────────────┘  │
│                                                              │
│  Network Policy: Pod-to-Pod traffic BLOCKED                  │
│  Network Policy: Internet access ALLOWED (for downloads)     │
│  Network Policy: Access to platform services BLOCKED         │
└──────────────────────────────────────────────────────────────┘
                          ↑
                          │ Ingress (HTTPS)
                          │
                   ┌──────────────┐
                   │   Student    │
                   │   Browser    │
                   └──────────────┘
```

### Взаимодействие с другими компонентами

**1. Backend Platform → Lab Deployment:**
- Студент нажимает "Запустить лабу"
- Backend Platform проверяет права доступа (ABAC)
- Backend Platform создаёт запись `LabInstance` в PostgreSQL (status=PENDING)
- Backend Platform публикует задачу в RabbitMQ

**2. Celery Worker → Lab Deployment:**
- Worker получает задачу из RabbitMQ
- Worker генерирует манифесты (Pod, Service, NetworkPolicy)
- Worker применяет манифесты через K8s API
- Worker ждёт Pod Ready (timeout 60 секунд)
- Worker обновляет `LabInstance` (status=RUNNING, connection_url)

**3. Student → Lab Deployment:**
- Студент получает `connection_url` от Backend Platform
- Студент открывает браузер и переходит по ссылке
- Ingress маршрутизирует запрос к Service → Pod
- Студент работает в изолированном окружении

**4. Cleanup → Lab Deployment:**
- Celery Beat проверяет timeout (каждые 5 минут)
- Worker удаляет Pod'ы, превысившие timeout (2 часа)
- Worker обновляет `LabInstance` (status=STOPPED)

---

## Жизненный цикл лабораторной работы

### Полный цикл от запроса до удаления

#### Этап 1: Запрос студента (Frontend → Backend Platform)

**Действия:**
1. Студент авторизован в системе (JWT токен от Zitadel)
2. Студент видит список доступных лаб курса
3. Студент нажимает "Запустить лабу: Nmap Scanning"

**HTTP запрос:**
```
POST /api/v1/lab-instances
Authorization: Bearer <jwt_token>
Body: {
  "lab_id": 123,
  "course_id": 456
}
```

**Backend Platform проверяет:**
- ✅ Студент назначен на курс (ABAC)
- ✅ Лаба одобрена (status=APPROVED)
- ✅ У студента нет активной лабы этого типа (бизнес-правило)
- ✅ Quota не исчерпана (namespace `labs` имеет свободные ресурсы)

**Результат:**
- Создаётся запись в PostgreSQL: `LabInstance(id=789, status=PENDING, student_id=123, lab_id=456)`
- Возвращается ответ: `{"lab_instance_id": 789, "status": "pending"}`

---

#### Этап 2: Постановка задачи (Backend Platform → RabbitMQ)

**Действия:**
Backend Platform публикует задачу `create_lab_pod` в RabbitMQ.

**Payload задачи:**
```json
{
  "task_type": "create_lab_pod",
  "lab_instance_id": 789,
  "student_id": 123,
  "lab_id": 456,
  "docker_image": "registry.gitlab.com/autolabs/labs/nmap:v1.0.0",
  "resources": {
    "cpu": "500m",
    "memory": "512Mi",
    "storage": "1Gi"
  },
  "network_isolated": true,
  "timeout_minutes": 120,
  "exposed_ports": [8080, 22],
  "env_vars": {
    "STUDENT_ID": "123",
    "LAB_NAME": "nmap-scanning"
  }
}
```

**Celery Task ID:** `abc123-def456-...`

Backend Platform сохраняет: `LabInstance.celery_task_id = "abc123..."`

---

#### Этап 3: Создание Pod'а (Celery Worker → Kubernetes API)

**Действия Celery Worker:**

**3.1. Генерация манифестов**

Worker использует Jinja2 шаблоны для генерации манифестов:

```python
# Псевдокод
pod_manifest = generate_pod_manifest(
    name=f"lab-instance-{lab_instance_id}",
    namespace="labs",
    docker_image="registry.gitlab.com/autolabs/labs/nmap:v1.0.0",
    labels={
        "lab_instance_id": "789",
        "student_id": "123",
        "lab_type": "nmap"
    },
    resources={"cpu": "500m", "memory": "512Mi"},
    security_context={"runAsNonRoot": True, "runAsUser": 1000}
)

service_manifest = generate_service_manifest(
    name=f"lab-instance-{lab_instance_id}-svc",
    namespace="labs",
    selector={"lab_instance_id": "789"},
    ports=[8080]
)

netpol_manifest = generate_networkpolicy_manifest(
    name=f"lab-instance-{lab_instance_id}-netpol",
    namespace="labs",
    pod_selector={"lab_instance_id": "789"},
    allow_internet=True
)
```

**3.2. Применение манифестов через K8s API**

```python
# Создание Pod'а
k8s_client.create_namespaced_pod(namespace="labs", body=pod_manifest)

# Ожидание Pod Ready (timeout 60 секунд)
await wait_for_pod_ready(pod_name="lab-instance-789", namespace="labs", timeout=60)

# Создание Service
k8s_client.create_namespaced_service(namespace="labs", body=service_manifest)

# Создание NetworkPolicy
k8s_client.create_namespaced_network_policy(namespace="labs", body=netpol_manifest)
```

**3.3. Получение connection details**

После успешного создания Pod'а:

```python
# Получение ClusterIP Service
service = k8s_client.read_namespaced_service(name="lab-instance-789-svc", namespace="labs")
cluster_ip = service.spec.cluster_ip

# Генерация connection_url через Ingress
connection_url = f"https://autolabs.example.com/labs/{lab_instance_id}"

# Или через NodePort (если используется)
# node_port = service.spec.ports[0].node_port
# connection_url = f"http://node-ip:{node_port}"
```

**3.4. Обновление базы данных**

```python
await db.execute(
    """
    UPDATE lab_instances
    SET status = 'running',
        k8s_pod_name = :pod_name,
        connection_url = :url,
        started_at = NOW()
    WHERE id = :id
    """,
    {
        "pod_name": "lab-instance-789",
        "url": "https://autolabs.example.com/labs/789",
        "id": 789
    }
)
```

**Время выполнения:** 10-30 секунд (зависит от скорости pull образа).

---

#### Этап 4: Работа студента (Student → Lab Pod)

**Действия:**

1. **Frontend опрашивает статус** (polling каждые 2 секунды):
   ```
   GET /api/v1/lab-instances/789
   Response: {"status": "running", "connection_url": "https://autolabs.example.com/labs/789"}
   ```

2. **Студент переходит по ссылке**:
   - Браузер открывает `https://autolabs.example.com/labs/789`
   - Ingress маршрутизирует запрос к Service `lab-instance-789-svc`
   - Service направляет трафик к Pod'у `lab-instance-789`

3. **Студент видит web-интерфейс лабы**:
   - Если лаба web-based (например, Jupyter Notebook, ttyd terminal)
   - Если лаба CLI-based (SSH или web-based terminal)
   - Если лаба GUI-based (noVNC для графических приложений)

4. **Студент выполняет задание**:
   - Запускает команды (например, `nmap -sV target.com`)
   - Анализирует результаты
   - Сохраняет вывод команд (копирует в отчёт)

**Важно:**
- Все данные в Pod'е эфемерны (не сохраняются после удаления)
- Студент должен скопировать результаты до завершения работы

---

#### Этап 5: Завершение работы (Student → Backend Platform)

**Вариант A: Студент вручную останавливает лабу**

```
POST /api/v1/lab-instances/789/stop
```

Backend Platform публикует задачу `delete_lab_pod` в RabbitMQ.

**Вариант B: Автоматическое удаление по timeout**

Celery Beat каждые 5 минут запускает задачу `cleanup_timeout_labs`:

```python
# Найти лабы, которые работают >2 часов
timeout_labs = await db.fetch_all(
    """
    SELECT id FROM lab_instances
    WHERE status = 'running'
    AND started_at < NOW() - INTERVAL '2 hours'
    """
)

# Для каждой лабы: постановка задачи delete_lab_pod
for lab in timeout_labs:
    celery_app.send_task('delete_lab_pod', kwargs={'lab_instance_id': lab.id, 'force': True})
```

---

#### Этап 6: Удаление ресурсов (Celery Worker → Kubernetes API)

**Действия Worker:**

```python
# Удаление NetworkPolicy
k8s_client.delete_namespaced_network_policy(
    name=f"lab-instance-{lab_instance_id}-netpol",
    namespace="labs"
)

# Удаление Service
k8s_client.delete_namespaced_service(
    name=f"lab-instance-{lab_instance_id}-svc",
    namespace="labs"
)

# Удаление Pod'а
k8s_client.delete_namespaced_pod(
    name=f"lab-instance-{lab_instance_id}",
    namespace="labs",
    grace_period_seconds=10  # Даём Pod'у 10 секунд на graceful shutdown
)

# Обновление базы данных
await db.execute(
    """
    UPDATE lab_instances
    SET status = 'stopped', stopped_at = NOW()
    WHERE id = :id
    """,
    {"id": lab_instance_id}
)
```

**Время выполнения:** 5-15 секунд.

**Результат:**
- Pod удалён из K8s
- Все ресурсы (CPU, RAM) освобождены
- Запись в PostgreSQL сохранена (для истории)

---

## Архитектура Pod'а для лабы

### Базовая структура Pod'а

**Минималистичный Pod (только контейнер с лабой):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab-instance-789
  namespace: labs
  labels:
    app: lab-pod
    lab_instance_id: "789"
    student_id: "123"
    lab_type: "nmap"
spec:
  # Security Context (запуск от non-root пользователя)
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000

  containers:
  - name: lab-container
    image: registry.gitlab.com/autolabs/labs/nmap:v1.0.0
    imagePullPolicy: Always

    # Resource limits (защита от resource exhaustion)
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"

    # Environment variables
    env:
    - name: STUDENT_ID
      value: "123"
    - name: LAB_INSTANCE_ID
      value: "789"
    - name: TIMEOUT_MINUTES
      value: "120"

    # Ports
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP

    # Security Context (container level)
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false  # Некоторые инструменты требуют запись
      capabilities:
        drop:
        - ALL
        add:
        - NET_RAW  # Для nmap, tcpdump (нужен для ICMP)

    # Liveness probe (перезапуск если контейнер завис)
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 30

  # Restart policy (не перезапускать автоматически)
  restartPolicy: Never

  # Tolerations (если нужно запускать на определённых нодах)
  tolerations:
  - key: "workload"
    operator: "Equal"
    value: "labs"
    effect: "NoSchedule"
```

### Обоснования архитектурных решений

**1. `runAsNonRoot: true` + `runAsUser: 1000`**

**Обоснование:**
- Запрет запуска от root пользователя (Pod Security Standards)
- Даже если студент получит shell в контейнере, он не имеет root привилегий
- Усложняет container escape атаки

**2. `restartPolicy: Never`**

**Обоснование:**
- Лаба — эфемерное окружение, не требует автоматического перезапуска
- Если Pod упал — студент видит ошибку и может запустить новую лабу
- Упрощает логику (нет бесконечных рестартов)

**3. `allowPrivilegeEscalation: false`**

**Обоснование:**
- Запрет получения дополнительных привилегий внутри контейнера
- Защита от escalation атак (setuid binaries)

**4. `capabilities: drop ALL + add NET_RAW`**

**Обоснование:**
- Удаляем все Linux capabilities по умолчанию
- Добавляем только `NET_RAW` (необходим для nmap, ping, tcpdump)
- Минимальный набор привилегий для работы сетевых инструментов

**5. `resources: limits`**

**Обоснование:**
- Защита от "жадных" лаб (студент не может съесть все ресурсы кластера)
- Гарантия fair share ресурсов между студентами

---

### Продвинутая структура Pod'а (с init containers и sidecars)

**Для сложных лаб (например, Metasploit с PostgreSQL):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab-instance-790
  namespace: labs
  labels:
    lab_instance_id: "790"
    student_id: "124"
    lab_type: "metasploit"
spec:
  # Init Container: Инициализация базы данных
  initContainers:
  - name: init-db
    image: postgres:14
    command:
    - sh
    - -c
    - |
      createdb metasploit_db
      psql -c "CREATE USER msf WITH PASSWORD 'msf123'"
    env:
    - name: POSTGRES_PASSWORD
      value: "postgres"

  containers:
  # Основной контейнер: Metasploit
  - name: metasploit
    image: registry.gitlab.com/autolabs/labs/metasploit:v1.0.0
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
    ports:
    - containerPort: 4444  # Metasploit handler
    - containerPort: 8080  # Web UI
    env:
    - name: DATABASE_URL
      value: "postgresql://msf:msf123@localhost:5432/metasploit_db"

  # Sidecar контейнер: PostgreSQL
  - name: postgres
    image: postgres:14
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1Gi"
    env:
    - name: POSTGRES_PASSWORD
      value: "postgres"
    - name: POSTGRES_DB
      value: "metasploit_db"
    volumeMounts:
    - name: postgres-data
      mountPath: /var/lib/postgresql/data

  volumes:
  - name: postgres-data
    emptyDir: {}  # Эфемерное хранилище (удаляется с Pod'ом)
```

**Обоснование использования sidecars:**

**Плюсы:**
- ✅ Единый Pod для связанных компонентов (Metasploit + PostgreSQL)
- ✅ Локальная коммуникация через localhost (быстро и безопасно)
- ✅ Автоматическое удаление всех компонентов одновременно

**Минусы:**
- ❌ Больше потребление ресурсов (PostgreSQL в каждом Pod'е)
- ❌ Сложнее templates

**Альтернатива:**
Общий PostgreSQL для всех Metasploit лаб (отдельный StatefulSet). Проблемы:
- Нужна изоляция баз данных (каждая лаба = отдельная БД)
- Сложнее cleanup (нужно удалять БД при удалении лабы)

**Рекомендация:** Для MVP использовать sidecar подход.

---

## Типы лабораторных работ

### 1. CLI-based лабы (командная строка)

**Примеры:** nmap, netcat, curl, sqlmap

**Архитектура:**
- Контейнер с предустановленными CLI инструментами
- Web-based terminal (ttyd или wetty) для доступа через браузер
- Или SSH доступ (требует NodePort или LoadBalancer)

**Типичный Docker образ:**
```dockerfile
FROM ubuntu:22.04

# Установка инструментов
RUN apt-get update && apt-get install -y \
    nmap \
    netcat \
    curl \
    dnsutils \
    iputils-ping \
    traceroute

# Установка web-based terminal (ttyd)
RUN wget https://github.com/tsl0922/ttyd/releases/download/1.7.3/ttyd.x86_64 \
    && mv ttyd.x86_64 /usr/local/bin/ttyd \
    && chmod +x /usr/local/bin/ttyd

# Non-root user
RUN useradd -m -u 1000 student
USER student

# Entrypoint: ttyd запускает bash
ENTRYPOINT ["ttyd", "-p", "8080", "bash"]
```

**Connection:** Студент открывает `https://autolabs.example.com/labs/789` → видит web terminal

**Обоснование:**
- ✅ Простота (не нужен SSH клиент)
- ✅ Работает из любого браузера
- ✅ Легко интегрируется с Ingress

---

### 2. Web-based лабы (веб-интерфейс)

**Примеры:** Jupyter Notebook, DVWA (Damn Vulnerable Web App), WebGoat

**Архитектура:**
- Контейнер с веб-приложением
- HTTP server слушает на порту 8080
- Ingress маршрутизирует трафик

**Типичный Docker образ (DVWA):**
```dockerfile
FROM php:7.4-apache

# Установка DVWA
RUN git clone https://github.com/digininja/DVWA /var/www/html/

# Настройка Apache
RUN chown -R www-data:www-data /var/www/html

# Non-root user (Apache уже работает от www-data)
EXPOSE 80
CMD ["apache2-foreground"]
```

**Connection:** Студент открывает `https://autolabs.example.com/labs/790` → видит DVWA интерфейс

**Обоснование:**
- ✅ Наиболее user-friendly (весь интерфейс в браузере)
- ✅ Не требует дополнительного ПО

---

### 3. GUI-based лабы (графический интерфейс)

**Примеры:** Wireshark, Burp Suite, Metasploit GUI

**Проблема:** GUI приложения требуют X11 server, который нельзя просто так запустить в браузере.

**Решение: noVNC (VNC через веб-браузер)**

**Архитектура:**
- Контейнер с X11 server (Xvfb — виртуальный framebuffer)
- VNC server (x11vnc или TigerVNC)
- noVNC (WebSocket прокси для VNC)
- GUI приложение (Wireshark)

**Типичный Docker образ:**
```dockerfile
FROM ubuntu:22.04

# Установка X11, VNC, noVNC
RUN apt-get update && apt-get install -y \
    xvfb \
    x11vnc \
    novnc \
    websockify \
    supervisor \
    wireshark

# Создание non-root пользователя
RUN useradd -m -u 1000 student

# Supervisor config (запуск всех сервисов)
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 8080
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

**supervisord.conf:**
```ini
[supervisord]
nodaemon=true

[program:xvfb]
command=/usr/bin/Xvfb :99 -screen 0 1024x768x24
autorestart=true

[program:x11vnc]
command=/usr/bin/x11vnc -display :99 -forever -shared
autorestart=true

[program:novnc]
command=/usr/share/novnc/utils/launch.sh --vnc localhost:5900 --listen 8080
autorestart=true

[program:wireshark]
command=/usr/bin/wireshark
environment=DISPLAY=:99
autorestart=true
user=student
```

**Connection:** Студент открывает `https://autolabs.example.com/labs/791` → видит Wireshark GUI через noVNC

**Обоснование:**
- ✅ Полноценный GUI в браузере (без установки VNC клиента)
- ✅ Работает на любом устройстве
- ❌ Повышенное потребление ресурсов (X11 + VNC)
- ❌ Может быть медленным при плохом интернете

**Это критичный вопрос для проработки:** Нужны ли GUI лабы в MVP?

---

## Networking и доступ студента

### Проблема: Как студент получает доступ к Pod'у в кластере?

**Варианты решений:**

### Вариант A: Ingress с path-based routing (рекомендуется)

**Архитектура:**
```
Student → https://autolabs.example.com/labs/789
          ↓
     Ingress NGINX
          ↓
     Service: lab-instance-789-svc (ClusterIP)
          ↓
     Pod: lab-instance-789
```

**Ingress манифест (динамически создаётся Celery Worker):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-instance-789-ingress
  namespace: labs
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: autolabs.example.com
    http:
      paths:
      - path: /labs/789
        pathType: Prefix
        backend:
          service:
            name: lab-instance-789-svc
            port:
              number: 8080
```

**Обоснование:**

**Плюсы:**
- ✅ Единый домен для всех лаб
- ✅ HTTPS из коробки (TLS termination в Ingress)
- ✅ Не требует дополнительных портов
- ✅ Path-based routing (легко идентифицировать лабу)

**Минусы:**
- ❌ Сложнее конфигурация для WebSocket (нужен для noVNC)
- ❌ Все лабы через один Ingress (potential bottleneck)

---

### Вариант B: NodePort (для on-premise без LoadBalancer)

**Архитектура:**
```
Student → http://node-ip:30789
          ↓
     K8s Node (NodePort 30789)
          ↓
     Service: lab-instance-789-svc (NodePort)
          ↓
     Pod: lab-instance-789
```

**Service манифест:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab-instance-789-svc
  namespace: labs
spec:
  type: NodePort
  selector:
    lab_instance_id: "789"
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30789  # Диапазон: 30000-32767
```

**Обоснование:**

**Плюсы:**
- ✅ Простота (не нужен Ingress)
- ✅ Прямой доступ к Pod'у

**Минусы:**
- ❌ Ограничен диапазон портов (30000-32767 = ~2700 портов)
- ❌ Нет HTTPS (нужен TLS в контейнере)
- ❌ Студент должен знать IP ноды

**Когда использовать:** Только для dev окружения или малого количества студентов.

---

### Вариант C: LoadBalancer (для облачных провайдеров)

**Архитектура:**
```
Student → http://external-ip:8080
          ↓
     Cloud Load Balancer (AWS ELB, GCP LB)
          ↓
     Service: lab-instance-789-svc (LoadBalancer)
          ↓
     Pod: lab-instance-789
```

**Service манифест:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab-instance-789-svc
  namespace: labs
spec:
  type: LoadBalancer
  selector:
    lab_instance_id: "789"
  ports:
  - port: 8080
    targetPort: 8080
```

**Обоснование:**

**Плюсы:**
- ✅ Автоматическое создание внешнего IP
- ✅ HA из коробки (облачный балансировщик)

**Минусы:**
- ❌ Стоимость (каждый LoadBalancer = $10-20/месяц)
- ❌ Не масштабируется для большого количества лаб

**Когда использовать:** Только если бюджет позволяет.

---

### Рекомендация: Вариант A (Ingress) для production

**Динамическое создание Ingress для каждой лабы:**

Celery Worker создаёт Ingress при создании Pod'а:

```python
def create_lab_ingress(lab_instance_id, service_name):
    ingress_manifest = {
        "apiVersion": "networking.k8s.io/v1",
        "kind": "Ingress",
        "metadata": {
            "name": f"lab-instance-{lab_instance_id}-ingress",
            "namespace": "labs",
            "annotations": {
                "nginx.ingress.kubernetes.io/rewrite-target": "/",
                "nginx.ingress.kubernetes.io/websocket-services": service_name  # Для noVNC
            }
        },
        "spec": {
            "rules": [{
                "host": "autolabs.example.com",
                "http": {
                    "paths": [{
                        "path": f"/labs/{lab_instance_id}",
                        "pathType": "Prefix",
                        "backend": {
                            "service": {
                                "name": service_name,
                                "port": {"number": 8080}
                            }
                        }
                    }]
                }
            }]
        }
    }

    k8s_client.create_namespaced_ingress(namespace="labs", body=ingress_manifest)
```

**Connection URL:** `https://autolabs.example.com/labs/{lab_instance_id}`

---

## Безопасность и изоляция

### Философия: Defence in Depth для студенческих Pod'ов

Студенческие Pod'ы — **наиболее уязвимая** часть системы, так как:
- Студенты запускают произвольный код
- Студенты могут пытаться выйти за пределы контейнера
- Студенты (теоретически) могут попытаться атаковать платформу

**Многоуровневая защита:**

### Уровень 1: Pod Security Standards

Namespace `labs` имеет **restricted** Pod Security Standard:

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

**Эффект:**
- ❌ Нельзя создать privileged Pod
- ❌ Нельзя использовать hostPath, hostNetwork, hostPID
- ❌ Нельзя запускать от root
- ❌ Нельзя добавлять dangerous capabilities

**Обоснование:**
Даже если Celery Worker скомпрометирован и пытается создать вредоносный Pod, K8s отклонит манифест.

---

### Уровень 2: SecurityContext в Pod манифесте

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault  # Seccomp profile для ограничения syscalls

  containers:
  - name: lab-container
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false  # Некоторые лабы требуют запись
      capabilities:
        drop:
        - ALL
        add:
        - NET_RAW  # Только для сетевых инструментов
```

**Обоснование:**
- `runAsNonRoot` — защита от root exploits
- `allowPrivilegeEscalation: false` — защита от setuid binaries
- `capabilities: drop ALL` — минимальные привилегии

---

### Уровень 3: Network Policies

**Default Deny для namespace `labs`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: labs
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**Allow Ingress from Ingress NGINX:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-from-nginx
  namespace: labs
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

**Allow Egress to Internet:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet
  namespace: labs
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

  # HTTP/HTTPS (для apt-get install, pip install)
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
```

**Критически важно: Pod-to-Pod traffic BLOCKED**

Студенческие Pod'ы НЕ могут общаться друг с другом. Даже если студент запустит port scanner внутри лабы, он не сможет просканировать другие лабы.

**Обоснование:**
Защита от lateral movement. Компрометация одной лабы не даёт доступ к другим.

---

### Уровень 4: Resource Quotas и LimitRanges

**Защита от resource exhaustion:**

```yaml
# LimitRange на namespace labs
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-limits
  namespace: labs
spec:
  limits:
  - max:
      cpu: "2"
      memory: 2Gi
    min:
      cpu: "100m"
      memory: 128Mi
    default:
      cpu: "500m"
      memory: 512Mi
    type: Container
```

**Обоснование:**
Студент не может запустить "жадную" лабу (fork bomb, memory leak), которая убьёт кластер.

---

### Уровень 5: Image Security

**Проблема:** Docker образы могут содержать уязвимости.

**Решение:**

1. **Image Scanning (Trivy):**
   ```bash
   trivy image registry.gitlab.com/autolabs/labs/nmap:v1.0.0
   ```
   Сканирование на уязвимости перед публикацией образа.

2. **Image Signing (Cosign):**
   ```bash
   cosign sign registry.gitlab.com/autolabs/labs/nmap:v1.0.0
   ```
   Подпись образов для защиты от подмены.

3. **Private Registry:**
   - Все образы в GitLab Container Registry (приватный)
   - RBAC на доступ к registry

**Это критичный вопрос для проработки.**

---

## Resource Management

### Стратегия распределения ресурсов

**Проблема:** Как балансировать между количеством студентов и качеством работы лаб?

### Категории лабораторных работ по ресурсам

**1. Light Labs (лёгкие):**
- Примеры: nmap, netcat, curl
- Resources: 250m CPU, 256Mi RAM
- Количество одновременно: до 100 студентов (на кластере с 25 CPU, 25Gi RAM)

**2. Medium Labs (средние):**
- Примеры: DVWA, WebGoat, basic Metasploit
- Resources: 500m CPU, 512Mi RAM
- Количество одновременно: до 50 студентов

**3. Heavy Labs (тяжёлые):**
- Примеры: Metasploit с PostgreSQL, Burp Suite, Wireshark GUI
- Resources: 1 CPU, 1Gi RAM
- Количество одновременно: до 25 студентов

### Динамическое распределение ресурсов

**Вариант A: Фиксированные ресурсы**

Каждая лаба имеет фиксированные requests/limits.

**Плюсы:**
- ✅ Предсказуемость
- ✅ Простота

**Минусы:**
- ❌ Неэффективное использование ресурсов (light labs не используют выделенные ресурсы)

---

**Вариант B: Vertical Pod Autoscaler (VPA)**

VPA автоматически корректирует requests/limits на основе реального потребления.

**Плюсы:**
- ✅ Эффективное использование ресурсов
- ✅ Автоматическая оптимизация

**Минусы:**
- ❌ VPA требует рестарт Pod'а (не подходит для эфемерных лаб)
- ❌ Сложность настройки

---

**Рекомендация:** Вариант A для MVP (фиксированные ресурсы).

---

### Квоты на студента

**Бизнес-правило:** Студент не может запустить более N лаб одновременно.

**Реализация в Backend Platform:**

```python
async def check_student_quota(student_id: int, max_concurrent_labs: int = 3):
    active_labs = await db.fetch_val(
        """
        SELECT COUNT(*) FROM lab_instances
        WHERE student_id = :student_id AND status = 'running'
        """,
        {"student_id": student_id}
    )

    if active_labs >= max_concurrent_labs:
        raise QuotaExceededError(f"You can have maximum {max_concurrent_labs} active labs")
```

**Обоснование:**
- Защита от abuse (студент не может "захватить" все ресурсы кластера)
- Fair share ресурсов между студентами

**Это критичный вопрос для проработки:** Какое значение max_concurrent_labs оптимально?

---

## Cleanup и автоматическое удаление

### Философия: Лабы — эфемерные окружения

Лабы должны автоматически удаляться после определённого времени, чтобы освободить ресурсы.

### Механизм timeout

**Celery Beat задача `cleanup_timeout_labs`** (запускается каждые 5 минут):

```python
@celery_app.task
def cleanup_timeout_labs(timeout_minutes: int = 120):
    """
    Удаление лаб, которые работают дольше timeout_minutes
    """
    # Найти лабы с превышением timeout
    timeout_labs = db.fetch_all(
        """
        SELECT id, lab_instance_id, student_id
        FROM lab_instances
        WHERE status = 'running'
        AND started_at < NOW() - INTERVAL ':timeout minutes'
        """,
        {"timeout": timeout_minutes}
    )

    # Для каждой лабы: постановка задачи delete_lab_pod
    for lab in timeout_labs:
        logger.info(f"Cleanup timeout lab: {lab.id}")
        celery_app.send_task(
            'delete_lab_pod',
            kwargs={
                'lab_instance_id': lab.id,
                'force': True,
                'reason': 'timeout'
            }
        )
```

**Обоснование:**
- Студенты забывают остановить лабы вручную
- Освобождение ресурсов для других студентов
- Предотвращение resource exhaustion

---

### Варианты timeout

**1. Фиксированный timeout для всех лаб:**
- Все лабы удаляются через 2 часа

**2. Кастомный timeout для каждого типа лабы:**
- Light labs: 1 час
- Medium labs: 2 часа
- Heavy labs: 3 часа

**3. Продление timeout студентом:**
- Студент может нажать "Продлить на 1 час"
- Максимум 2 продления

**Рекомендация:** Вариант 1 для MVP (2 часа для всех лаб).

---

### Warning перед удалением

**UX улучшение:**

За 10 минут до удаления отправить уведомление студенту:

```python
# В Celery Worker перед удалением
if time_remaining < 10 * 60:  # 10 минут
    await notification_service.send_warning(
        student_id=student_id,
        message="Your lab will be deleted in 10 minutes. Please save your work."
    )
```

**Это критичный вопрос для проработки:** Как отправлять уведомления? (Email, WebSocket, Push notifications?)

---

## Ключевые вопросы для проработки

Эти вопросы требуют детального анализа и тестирования:

### 1. **Как студент получает доступ к лабе?**

**Варианты:**
- Ingress с path-based routing (рекомендуется)
- NodePort (для малых окружений)
- LoadBalancer (дорого для множества лаб)

**Что нужно проработать:**
- Динамическое создание Ingress для каждой лабы
- WebSocket поддержка для noVNC
- TLS termination в Ingress

---

### 2. **Нужен ли persistent storage для лабораторных работ?**

**Проблема:** Студент потеряет все данные после удаления Pod'а.

**Вариант A: Эфемерное хранилище (рекомендуется для MVP)**
- Все данные в emptyDir (удаляются с Pod'ом)
- Студент должен скопировать результаты до завершения работы

**Вариант B: PersistentVolume для каждой лабы**
- Данные сохраняются между запусками
- Студент может "заморозить" состояние лабы

**Вопросы:**
- Как управлять PV после завершения лабы? (удалять или сохранять?)
- Стоимость хранения (если 50 студентов x 10 лаб x 1 GB = 500 GB)

**Рекомендация:** Эфемерное хранилище для MVP. Persistent storage — опция для будущего.

---

### 3. **GUI приложения: noVNC или альтернативы?**

**Проблема:** Wireshark, Burp Suite требуют GUI.

**Вариант A: noVNC (VNC через браузер)**
- ✅ Работает в любом браузере
- ❌ Высокое потребление ресурсов (X11 + VNC)
- ❌ Может быть медленным

**Вариант B: X11 forwarding через SSH**
- ✅ Нативная производительность
- ❌ Требует X11 server на клиенте (не работает на Windows без дополнительного ПО)

**Вариант C: Web-based альтернативы**
- Использовать tshark (CLI Wireshark) вместо GUI
- Использовать web-based Burp Suite альтернативы

**Вопросы:**
- Сколько лаб реально требуют GUI? (может быть, их меньшинство?)
- Можно ли переделать GUI лабы в CLI-based?

**Это критичный вопрос:** Нужны ли GUI лабы в MVP?

---

### 4. **Timeout механизм и продление работы**

**Вопросы:**
- Какой оптимальный timeout? (1 час, 2 часа, 4 часа?)
- Разрешить ли студентам продлевать timeout?
- Сколько максимум продлений?
- Как уведомлять студента о скором удалении?

**Рекомендация:**
- MVP: 2 часа фиксированный timeout, без продлений
- Production: кастомный timeout + возможность продления

---

### 5. **Квоты на количество одновременных лаб для студента**

**Бизнес-правило:** Студент не может запустить более N лаб одновременно.

**Вопросы:**
- Какое оптимальное значение N? (1, 3, 5?)
- Разные квоты для разных ролей? (Student: 2, Teacher: 5, Admin: unlimited?)

**Рекомендация:** N=3 для студентов (баланс между удобством и fair share).

---

### 6. **Логирование действий студентов в лабах**

**Проблема:** Антиплагиат — как доказать, что студент сам выполнил работу?

**Вариант A: Логирование команд**
- Все команды студента логируются в файл
- При завершении лабы файл отправляется в PostgreSQL

**Вариант B: Без логирования**
- Доверие студенту
- Проверка через отчёты

**Вопросы:**
- Этично ли логировать все действия студента?
- Как хранить логи? (PostgreSQL, S3, отдельный сервис?)

**Это критичный вопрос для проработки.**

---

### 7. **Image Registry и версионирование лаб**

**Проблема:** Как управлять версиями Docker образов лаб?

**Вопросы:**
- Использовать теги (nmap:v1.0.0, nmap:v1.1.0) или immutable digests?
- Как откатываться к старой версии лабы?
- Как обновлять лабы без breaking changes?

**Рекомендация:**
- Использовать семантическое версионирование (nmap:v1.0.0)
- В PostgreSQL хранить docker_image с конкретной версией

---

## Заключение

### Роль Lab Deployments в платформе AutoLabs

Lab Deployments — это **сердце** платформы. Вся инфраструктура (Backend Platform, Celery Workers, Kubernetes, Network Policies, RBAC) существует ради одной цели:

**Дать студенту безопасное, изолированное, временное окружение для практики по кибербезопасности.**

### Ключевые архитектурные принципы

1. **Ephemeral Pods (эфемерные окружения)**
   - Каждая лаба — временный Pod
   - Автоматическое удаление по timeout
   - Чистое состояние при каждом запуске

2. **Defence in Depth для безопасности**
   - 5 уровней защиты (Pod Security Standards, SecurityContext, Network Policies, Resource Quotas, Image Security)
   - Изоляция Pod-to-Pod (студенты не видят друг друга)
   - Non-root пользователи, минимальные capabilities

3. **Resource Management**
   - LimitRange + ResourceQuota на namespace `labs`
   - Категории лаб (light, medium, heavy)
   - Квоты на студента (максимум N одновременных лаб)

4. **Простота доступа**
   - Web-based интерфейс (CLI, Web, GUI через noVNC)
   - Ingress с path-based routing
   - Один URL для доступа

5. **Автоматизация жизненного цикла**
   - Celery Workers создают Pod'ы
   - Celery Beat удаляет timeout лабы
   - Backend Platform отслеживает статусы

### Типы лабораторных работ

1. **CLI-based** (nmap, netcat) — web terminal (ttyd)
2. **Web-based** (DVWA, WebGoat) — HTTP интерфейс
3. **GUI-based** (Wireshark, Burp Suite) — noVNC (требует проработки)

### Критические вопросы требующие проработки

1. **Как студент получает доступ?** (Ingress vs NodePort vs LoadBalancer)
2. **Persistent storage?** (эфемерное vs PersistentVolume)
3. **GUI приложения?** (noVNC vs X11 forwarding vs альтернативы)
4. **Timeout механизм** (длительность, продление, уведомления)
5. **Квоты на студента** (сколько одновременных лаб?)
6. **Логирование действий** (антиплагиат, этичность)
7. **Image versioning** (управление версиями Docker образов)

### Рекомендации для следующих шагов

**Этап 1: MVP**
- CLI-based и Web-based лабы (без GUI)
- Ingress с path-based routing
- Эфемерное хранилище (emptyDir)
- Фиксированный timeout 2 часа
- Квота: 3 одновременные лабы на студента
- Без логирования действий

**Этап 2: Production**
- Добавить GUI лабы через noVNC
- Persistent storage (опционально для студента)
- Продление timeout
- Логирование для антиплагиата
- Push notifications о скором удалении

**Этап 3: Advanced**
- Snapshot состояния лабы (сохранение прогресса)
- Sharing лаб между студентами (для командной работы)
- Live collaboration (несколько студентов в одной лабе)

### Финальная мысль

Lab Deployments — это компромисс между:
- **Безопасностью** (изоляция, ограничения)
- **Удобством** (простота доступа, GUI)
- **Стоимостью** (ресурсы кластера)

Нет универсального решения. Каждое архитектурное решение требует анализа trade-offs и тестирования в реальных условиях.

---

## Дополнительные ресурсы

- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Ingress NGINX](https://kubernetes.github.io/ingress-nginx/)
- [noVNC Documentation](https://novnc.com/)
- [ttyd (web terminal)](https://github.com/tsl0922/ttyd)
- [Trivy Image Scanning](https://trivy.dev/)

---

**Поздравляем! Документация платформы AutoLabs завершена. 🎉**

Все 8 этапов документации созданы:
- ✅ 01-infrastructure
- ✅ 02-backend-platform
- ✅ 03-celery-workers
- ✅ 04-gitlab-cicd
- ✅ 05-kubernetes
- ✅ 06-authorization
- ✅ 07-lab-deployments
- ✅ 08-network-flows

**Платформа готова к детальной проработке и реализации!**
