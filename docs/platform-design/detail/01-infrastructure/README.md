# Инфраструктура (Infrastructure)

## О компоненте

### Что это?

**Инфраструктура AutoLabs** — это физический и виртуальный уровень платформы, на котором разворачиваются все компоненты системы.

**ВАЖНО:** Инфраструктура может быть развёрнута тремя способами:

1. **On-premise (Proxmox VE)** — на собственном сервере или bare-metal аренде
2. **Гибридный подход** — часть on-premise, часть в облаке (managed сервисы)
3. **Полностью облачный** — всё в облаке (AWS, GCP, Azure, Yandex Cloud)

**Базовый вариант (документирован далее):** **Proxmox VE** — гипервизор с открытым исходным кодом, позволяющий управлять виртуальными машинами (VM) и контейнерами LXC.

Инфраструктура (on-premise вариант) включает:
- **Proxmox VE** — гипервизор типа 1 (bare-metal)
- **3 виртуальные машины** — GitLab, Database, Kubernetes
- **Terraform** — Infrastructure as Code (IaC) для автоматизации (поддерживает все варианты)
- **Единая сеть** — простая конфигурация для MVP
- **PostgreSQL** — единый инстанс с множественными базами данных

**Гибридные и облачные варианты** позволяют:
- ✅ Вынести PostgreSQL на managed сервисы (AWS RDS, Yandex Managed PostgreSQL) — освобождает 4 CPU, 8 GB RAM
- ✅ Использовать GitLab.com (SaaS) вместо self-hosted — освобождает 4 CPU, 8 GB RAM
- ✅ Managed Kubernetes (EKS, GKE, Yandex K8s) вместо self-hosted
- ✅ S3/Object Storage вместо Minio — освобождает ресурсы K8s кластера

Детальное сравнение всех вариантов см. в разделе **"Сценарии развёртывания"** ниже.

### Где разворачивается?

- **Proxmox VE:** Устанавливается на физический сервер (bare-metal)
- **Виртуальные машины:** Создаются внутри Proxmox с использованием Terraform

### Ресурсы

#### Общие ресурсы сервера

| Параметр | Минимум (MVP) | Рекомендуемо |
|----------|---------------|--------------|
| **CPU** | 12 cores | 24 cores |
| **RAM** | 16 GB | 32 GB |
| **Диск** | 500 GB SSD | 1 TB SSD/NVMe |
| **Сеть** | 1 Гбит/с | 10 Гбит/с |

#### Распределение ресурсов по VM

| VM | CPU | RAM | Диск | Назначение |
|----|-----|-----|------|------------|
| **gitlab-vm** | 4 cores | 8 GB | 100 GB | GitLab CE + Container Registry + GitLab Runner |
| **database-vm** | 4 cores | 8 GB | 200 GB | PostgreSQL + Redis + Minio |
| **k8s-vm** | 4-16 cores | 8-16 GB | 200 GB | Single-node Kubernetes cluster |
| **Proxmox (хост)** | ~2 cores | ~2 GB | 50 GB | Overhead для гипервизора |

**Итого:** 12-24 CPU, 16-32 GB RAM, 500-550 GB диск

### Зависимости

Инфраструктура является **базовым уровнем** платформы. От неё зависят все остальные компоненты:

**On-premise вариант (Proxmox):**
1. **GitLab VM** → GitLab CE, GitLab Runner, Container Registry
2. **Database VM** → PostgreSQL (все базы данных), Redis, Minio
3. **Kubernetes VM** → все микросервисы платформы (Backend Platform, Celery Workers, Zitadel, RabbitMQ)

**Гибридный/облачный вариант:**
1. **GitLab** → GitLab.com (SaaS) или Managed GitLab
2. **Database** → Managed PostgreSQL (AWS RDS / Yandex Cloud / GCP Cloud SQL)
3. **Kubernetes** → Managed K8s (EKS / GKE / AKS / Yandex K8s) или self-hosted
4. **Storage** → S3 / Object Storage вместо Minio
5. **Cache** → Managed Redis (ElastiCache / Yandex Cache)

**Внешние зависимости (on-premise):**
- Физический сервер с установленным Proxmox VE (или bare-metal аренда)
- Доступ к интернету для загрузки образов ОС (Ubuntu 22.04 LTS)
- Terraform CLI для автоматизации создания VM

**Внешние зависимости (облачный/гибридный):**
- Аккаунт облачного провайдера (AWS / GCP / Azure / Yandex Cloud)
- API ключи для Terraform
- Terraform CLI для автоматизации создания ресурсов
- (Опционально) VPN или настройка публичных endpoints для связи on-premise ↔ облако

---

## Бизнес-логика и назначение

### Зачем нужна инфраструктура?

Инфраструктура — это **фундамент** платформы AutoLabs. Она обеспечивает:

1. **Изоляцию компонентов:** Разделение GitLab, Database и Kubernetes на отдельные VM повышает надёжность и упрощает управление
2. **Гибкость масштабирования:** Возможность увеличить ресурсы конкретной VM без влияния на другие
3. **Упрощение восстановления:** В случае сбоя одной VM остальные продолжают работать
4. **Независимость от облачных провайдеров:** Полный контроль над инфраструктурой, отсутствие ежемесячных платежей
5. **Автоматизация через Terraform:** Воспроизводимая инфраструктура "как код"

### Вклад в работу платформы

```
┌──────────────────────────────────────────────────────────┐
│                    Физический сервер                      │
│                     Proxmox VE                            │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  GitLab VM   │  │ Database VM  │  │ Kubernetes VM │  │
│  ├──────────────┤  ├──────────────┤  ├───────────────┤  │
│  │ • GitLab CE  │  │ • PostgreSQL │  │ • Backend     │  │
│  │ • Registry   │  │ • Redis      │  │ • Celery      │  │
│  │ • Runner     │  │ • Minio      │  │ • Zitadel     │  │
│  │              │  │              │  │ • RabbitMQ    │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│                                                            │
│            Управляется через Terraform                     │
└──────────────────────────────────────────────────────────┘
```

**Роли компонентов:**

- **Proxmox VE:** Гипервизор, управляющий виртуализацией
- **Terraform:** Автоматизация создания и конфигурации VM
- **GitLab VM:** CI/CD, контейнеры, исходный код
- **Database VM:** Хранение данных, кэширование, объекты
- **Kubernetes VM:** Оркестрация микросервисов платформы

---

## Обоснование выбора

### Почему Proxmox VE?

**Преимущества:**

1. **Открытый исходный код:** Бесплатная лицензия, нет vendor lock-in
2. **Web UI:** Удобный интерфейс управления VM и контейнерами
3. **KVM/LXC поддержка:** Поддержка полноценных VM и лёгких контейнеров
4. **REST API:** Интеграция с Terraform через провайдер `telmate/proxmox`
5. **Snapshot'ы:** Встроенная система резервного копирования
6. **Сообщество:** Большое комьюнити, много документации

**Недостатки:**

- Требует физический сервер или bare-metal аренду
- Менее зрелая экосистема по сравнению с VMware vSphere

### Альтернативы

| Решение | Плюсы | Минусы | Вывод |
|---------|-------|--------|-------|
| **VMware vSphere** | Промышленный стандарт, богатая экосистема | Платная лицензия (от $5000), vendor lock-in | ❌ Не подходит для бюджетного проекта |
| **OpenStack** | Полноценная облачная платформа | Сложная установка, требует 3+ узлов | ❌ Избыточно для MVP |
| **Облачный провайдер (AWS/GCP)** | Не нужно управлять железом | Ежемесячные платежи ($200-500/мес), внешняя зависимость | ⚠️ Подходит для production, дорого для учебного проекта |
| **VPS аренда (Hetzner)** | Дешевле облака ($50-100/мес), простая настройка | Меньше контроля, нет веб-интерфейса | ⚠️ Альтернатива для тех, у кого нет физического сервера |
| **Proxmox VE** | Бесплатный, гибкий, REST API, snapshot'ы | Требует физический сервер | ✅ **Оптимально для учебного проекта** |

### Почему Terraform?

**Преимущества:**

1. **Infrastructure as Code (IaC):** Инфраструктура хранится в виде кода (`.tf` файлы)
2. **Декларативный синтаксис:** Описываем желаемое состояние, Terraform сам создаёт ресурсы
3. **Воспроизводимость:** Можно пересоздать инфраструктуру за 5-10 минут
4. **Версионирование:** `.tf` файлы хранятся в Git, полная история изменений
5. **Провайдер Proxmox:** `telmate/proxmox` позволяет управлять VM через Terraform
6. **Plan/Apply:** Безопасное применение изменений с предварительным просмотром

**Критически важно для учебного проекта:**
- Преподаватели смогут быстро развернуть копию инфраструктуры
- Студенты увидят пример современного подхода к управлению инфраструктурой
- Снижение ошибок при ручной настройке

### Альтернативы Terraform

| Решение | Плюсы | Минусы | Вывод |
|---------|-------|--------|-------|
| **Ansible** | Универсальный, поддержка configuration management | Императивный подход, нет state management | ⚠️ Лучше для настройки ОС, хуже для создания VM |
| **Pulumi** | Использование Python/TypeScript вместо HCL | Меньше провайдеров, сложнее для новичков | ❌ Избыточно для простой инфраструктуры |
| **Ручное создание VM** | Не требует изучения новых инструментов | Не воспроизводимо, ошибки, долго | ❌ **Недопустимо для современного подхода** |
| **Terraform** | IaC стандарт, декларативный, plan/apply, провайдер Proxmox | Требует изучения HCL | ✅ **ОБЯЗАТЕЛЬНО для проекта** |

---

## Сценарии развёртывания: On-premise vs Гибрид vs Облако

**КРИТИЧЕСКИ ВАЖНО:** AutoLabs можно развернуть тремя способами в зависимости от доступных ресурсов и бюджета. Terraform поддерживает все три сценария, позволяя описать инфраструктуру единообразно.

### Сценарий A: On-premise (Proxmox) ✅

**Выбран для базовой документации**

```
Всё разворачивается на собственном сервере:

┌─────────────────────────────────────┐
│     Физический сервер (Proxmox)     │
├─────────────────────────────────────┤
│ • GitLab VM (self-hosted)           │
│ • PostgreSQL VM (self-hosted)       │
│ • Kubernetes VM (self-hosted)       │
└─────────────────────────────────────┘

Terraform Provider: telmate/proxmox
```

**Когда использовать:**
- ✅ Есть физический сервер или bare-metal аренда
- ✅ Нужен полный контроль над инфраструктурой
- ✅ Нет бюджета на облачные сервисы
- ✅ Учебный проект без критичных SLA

**Стоимость:** Только электричество + интернет (или ~$50-100/мес за bare-metal аренду)

---

### Сценарий B: Гибридный подход ⚠️

**Рекомендуется для production**

Комбинация on-premise и облачных managed сервисов для оптимизации ресурсов:

```
┌──────────────────────────────────────────────────────┐
│  On-premise (Proxmox или VPS)                        │
├──────────────────────────────────────────────────────┤
│  • Kubernetes VM (self-hosted)                       │
│    └─ Backend Platform, Celery Workers, Zitadel      │
└──────────────────────────────────────────────────────┘
                      ↓ ↑
┌──────────────────────────────────────────────────────┐
│  Облачные Managed сервисы                            │
├──────────────────────────────────────────────────────┤
│  • GitLab.com (SaaS) — бесплатный план               │
│  • Managed PostgreSQL (AWS RDS / Yandex Cloud)       │
│  • Managed Redis (AWS ElastiCache / Yandex Cache)    │
│  • S3 Storage (AWS S3 / Yandex Object Storage)       │
└──────────────────────────────────────────────────────┘

Terraform Providers:
  - telmate/proxmox (для K8s VM)
  - hashicorp/aws (для RDS, ElastiCache, S3)
  - yandex-cloud/yandex (для Managed PostgreSQL)
```

**Преимущества:**
- ✅ **Снижение нагрузки на память/CPU:** База данных в облаке освобождает ~4 CPU и 8 GB RAM
- ✅ **Автоматические бэкапы:** Managed сервисы делают бэкапы автоматически
- ✅ **Высокая доступность:** Managed PostgreSQL обычно имеет HA out-of-the-box
- ✅ **Масштабирование:** Можно увеличить размер БД без остановки сервиса
- ✅ **Меньше управления:** Не нужно настраивать PostgreSQL, Redis, мониторинг

**Недостатки:**
- ⚠️ Ежемесячная оплата за managed сервисы
- ⚠️ Зависимость от облачного провайдера
- ⚠️ Необходимость настройки сетевого взаимодействия (VPN или публичные endpoints)

**Стоимость:** ~$30-80/мес в зависимости от провайдера

**Когда использовать:**
- Есть небольшой бюджет ($30-80/мес)
- Нужна высокая доступность БД
- Ограниченные ресурсы на сервере (например, 12 CPU / 16 GB RAM)
- Production окружение с SLA требованиями

#### Примеры комбинаций:

**Вариант B1: GitLab SaaS + Managed PostgreSQL**
```
On-premise:
  ✅ Kubernetes (Backend, Celery, Zitadel, RabbitMQ)

Облако:
  ✅ GitLab.com Free Tier (бесплатно, 5 GB storage)
  ✅ Managed PostgreSQL (AWS RDS / Yandex Cloud)
  ✅ Managed Redis
  ✅ S3 Storage (для Minio замена)

Экономия ресурсов: -8 CPU, -16 GB RAM
Стоимость: ~$30-50/мес
```

**Вариант B2: Только Managed PostgreSQL**
```
On-premise:
  ✅ Kubernetes
  ✅ GitLab VM (self-hosted)
  ✅ Redis, Minio (в K8s)

Облако:
  ✅ Managed PostgreSQL только

Экономия ресурсов: -4 CPU, -8 GB RAM (освобождается database-vm)
Стоимость: ~$20-40/мес
```

**Вариант B3: GitLab + Database в облаке, только K8s on-premise**
```
On-premise:
  ✅ Kubernetes VM только (Backend Platform + Celery Workers)

Облако:
  ✅ GitLab.com или Managed GitLab
  ✅ Managed PostgreSQL
  ✅ Managed Redis
  ✅ S3 Storage

Экономия ресурсов: -12 CPU, -24 GB RAM
Стоимость: ~$60-100/мес
```

---

### Сценарий C: Полностью облачный ☁️

**Для тех, у кого нет физического сервера**

Всё разворачивается в облаке с использованием managed сервисов:

```
┌──────────────────────────────────────────────────────┐
│  Облачный провайдер (AWS / GCP / Azure / Yandex)     │
├──────────────────────────────────────────────────────┤
│  • Managed Kubernetes (EKS / GKE / AKS / YC K8s)     │
│    └─ Backend Platform, Celery Workers, Zitadel      │
│  • Managed PostgreSQL (RDS / Cloud SQL / Managed DB) │
│  • Managed Redis (ElastiCache / Memorystore / Cache) │
│  • S3 Storage (S3 / GCS / Blob / Object Storage)     │
│  • GitLab.com (SaaS) или Managed GitLab             │
└──────────────────────────────────────────────────────┘

Terraform Providers:
  - hashicorp/aws (для AWS)
  - hashicorp/google (для GCP)
  - hashicorp/azurerm (для Azure)
  - yandex-cloud/yandex (для Yandex Cloud)
```

**Преимущества:**
- ✅ **Не нужен физический сервер**
- ✅ **Максимальная отказоустойчивость:** Все компоненты managed с HA
- ✅ **Автоматическое масштабирование:** K8s Cluster Autoscaler, RDS auto-scaling
- ✅ **Нулевое управление инфраструктурой:** Провайдер управляет всем
- ✅ **Глобальная доступность:** Развернуть в любом регионе мира

**Недостатки:**
- ⚠️ **Высокая стоимость:** $150-300/мес для production setup
- ⚠️ **Vendor lock-in:** Привязка к конкретному облаку
- ⚠️ **Меньше контроля:** Ограничения managed сервисов

**Стоимость:** ~$150-300/мес в зависимости от нагрузки и провайдера

**Когда использовать:**
- Нет физического сервера
- Критичные SLA (99.9% uptime)
- Глобальная аудитория (нужны multiple regions)
- Коммерческий проект с бюджетом

---

## Детальное сравнение компонентов: Self-hosted vs Managed

### GitLab: Self-hosted vs SaaS vs Managed

| Вариант | Описание | Ресурсы | Стоимость | Управление | Вывод |
|---------|----------|---------|-----------|------------|-------|
| **Self-hosted GitLab CE** | Развернуть GitLab на VM в Proxmox | 4 CPU, 8 GB RAM, 100 GB disk | $0 (только сервер) | Полное (обновления, бэкапы, мониторинг) | ✅ **Оптимально для MVP** |
| **GitLab.com Free** | SaaS от GitLab (бесплатный план) | 0 (облако) | $0 (5 GB storage, 400 CI/CD минут/мес) | Нулевое | ✅ **Для тех, у кого мало ресурсов** |
| **GitLab.com Premium** | SaaS с расширенными функциями | 0 (облако) | $29/user/мес | Нулевое | ⚠️ Дорого для учебного проекта |
| **Managed GitLab** | AWS/GCP marketplace GitLab | Managed | $50-100/мес | Частичное (только настройка) | ⚠️ Для enterprise |

**Рекомендация:**
- **MVP:** Self-hosted GitLab CE на Proxmox VM
- **Production (ограниченный бюджет):** GitLab.com Free (если хватает лимитов)
- **Production (большой бюджет):** Self-hosted GitLab с дополнительными runner'ами

**Terraform поддержка:**
- Self-hosted: `proxmox_vm_qemu` (создать VM) + Ansible для установки
- GitLab.com: Настраивается вручную через Web UI, репозитории создаются через GitLab API

---

### PostgreSQL: Self-hosted vs Managed

| Вариант | Описание | Ресурсы | Стоимость/мес | Управление | HA | Вывод |
|---------|----------|---------|---------------|------------|-------|-------|
| **Self-hosted PostgreSQL** | PostgreSQL на VM в Proxmox | 4 CPU, 8 GB RAM, 200 GB | $0 | Полное | ❌ (single instance) | ✅ **MVP** |
| **AWS RDS PostgreSQL** | Managed PostgreSQL от AWS | db.t3.medium (2 vCPU, 4 GB) | ~$50-70 | Минимальное | ✅ Multi-AZ | ⚠️ Для production |
| **Google Cloud SQL** | Managed PostgreSQL от GCP | db-n1-standard-1 (1 vCPU, 3.75 GB) | ~$40-60 | Минимальное | ✅ HA | ⚠️ Для production |
| **Yandex Managed PostgreSQL** | Managed PostgreSQL от Яндекса | s2.micro (2 vCPU, 8 GB) | ~$20-40 | Минимальное | ✅ HA | ✅ **Гибридный подход (дешевле AWS)** |
| **Hetzner Cloud DB** | Managed PostgreSQL от Hetzner | CPX11 (2 vCPU, 2 GB) | ~$15-25 | Минимальное | ❌ | ⚠️ Нет HA в базовом плане |
| **DigitalOcean Managed DB** | Managed PostgreSQL от DO | db-s-1vcpu-1gb | ~$15 | Минимальное | ✅ HA (за доплату) | ⚠️ Для небольших проектов |

**Преимущества Managed PostgreSQL:**
- ✅ Автоматические бэкапы (point-in-time recovery)
- ✅ Высокая доступность (Multi-AZ или replica)
- ✅ Автоматическое масштабирование storage
- ✅ Мониторинг и алерты из коробки
- ✅ Автоматические обновления PostgreSQL

**Рекомендация:**
- **MVP:** Self-hosted PostgreSQL на Proxmox
- **Production (бюджет есть):** Yandex Managed PostgreSQL (~$25/мес) или AWS RDS (~$50/мес)
- **Production (большой бюджет):** AWS RDS Multi-AZ с read replicas

**Terraform конфигурация для Managed PostgreSQL:**

```hcl
# AWS RDS PostgreSQL
resource "aws_db_instance" "autolabs_postgres" {
  identifier             = "autolabs-postgres"
  engine                 = "postgres"
  engine_version         = "15.4"
  instance_class         = "db.t3.medium"
  allocated_storage      = 100
  storage_type           = "gp3"

  db_name  = "autolabs"
  username = "admin"
  password = var.db_password

  multi_az               = true  # High Availability
  backup_retention_period = 7
  backup_window          = "03:00-04:00"

  vpc_security_group_ids = [aws_security_group.db.id]

  tags = {
    Environment = "production"
    Project     = "AutoLabs"
  }
}

# Yandex Cloud Managed PostgreSQL
resource "yandex_mdb_postgresql_cluster" "autolabs_postgres" {
  name        = "autolabs-postgres"
  environment = "PRODUCTION"
  network_id  = yandex_vpc_network.autolabs.id

  config {
    version = "15"
    resources {
      resource_preset_id = "s2.micro"  # 2 vCPU, 8 GB RAM
      disk_type_id       = "network-ssd"
      disk_size          = 100
    }
  }

  host {
    zone      = "ru-central1-a"
    subnet_id = yandex_vpc_subnet.autolabs_a.id
  }

  host {
    zone      = "ru-central1-b"  # HA replica
    subnet_id = yandex_vpc_subnet.autolabs_b.id
  }
}
```

---

### Kubernetes: Self-hosted vs Managed

| Вариант | Описание | Ресурсы | Стоимость/мес | Управление | HA | Вывод |
|---------|----------|---------|---------------|------------|-------|-------|
| **Self-hosted K8s (single-node)** | K8s на VM в Proxmox (kubeadm) | 4-16 CPU, 8-16 GB RAM | $0 | Полное | ❌ | ✅ **MVP** |
| **Self-hosted K8s (multi-node)** | 3 masters + 2 workers в Proxmox | 20+ CPU, 40+ GB RAM | $0 (или bare-metal аренда) | Полное | ✅ | ⚠️ Production on-premise |
| **AWS EKS** | Managed Kubernetes от AWS | Control Plane: managed, Nodes: EC2 | ~$75 (control) + $50-150 (nodes) = $125-225 | Частичное (управление nodes) | ✅ | ⚠️ Production (AWS) |
| **Google GKE** | Managed Kubernetes от Google | Control Plane: managed, Nodes: GCE | ~$75 (control) + $40-120 (nodes) = $115-195 | Частичное | ✅ | ⚠️ Production (GCP) |
| **Azure AKS** | Managed Kubernetes от Azure | Control Plane: бесплатно, Nodes: VM | $0 (control) + $50-150 (nodes) = $50-150 | Частичное | ✅ | ⚠️ Production (Azure) |
| **Yandex Managed K8s** | Managed Kubernetes от Яндекса | Control Plane: managed, Nodes: VM | ~$30 (control) + $30-80 (nodes) = $60-110 | Частичное | ✅ | ✅ **Дешевле для РФ** |
| **DigitalOcean K8s** | Managed Kubernetes от DO | Control Plane: бесплатно, Nodes: Droplets | $0 (control) + $40-120 (nodes) = $40-120 | Частичное | ✅ | ⚠️ Для небольших проектов |

**Преимущества Managed Kubernetes:**
- ✅ Автоматические обновления K8s
- ✅ HA control plane из коробки
- ✅ Интеграция с облачными балансировщиками (LoadBalancer service)
- ✅ Автоматическое масштабирование узлов (Cluster Autoscaler)
- ✅ Managed add-ons (мониторинг, логирование)

**Рекомендация:**
- **MVP:** Self-hosted single-node K8s на Proxmox
- **Production (ограниченный бюджет):** Self-hosted multi-node K8s на Proxmox
- **Production (есть бюджет):** Yandex Managed K8s (~$60/мес) или Azure AKS (~$50/мес)
- **Production (большой бюджет):** AWS EKS или Google GKE

**Terraform конфигурация для Managed Kubernetes:**

```hcl
# AWS EKS
resource "aws_eks_cluster" "autolabs" {
  name     = "autolabs-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.28"

  vpc_config {
    subnet_ids = [
      aws_subnet.private_a.id,
      aws_subnet.private_b.id,
    ]
  }
}

resource "aws_eks_node_group" "autolabs_nodes" {
  cluster_name    = aws_eks_cluster.autolabs.name
  node_group_name = "autolabs-nodes"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = [aws_subnet.private_a.id, aws_subnet.private_b.id]

  scaling_config {
    desired_size = 2
    max_size     = 5
    min_size     = 1
  }

  instance_types = ["t3.medium"]  # 2 vCPU, 4 GB RAM
}

# Yandex Managed Kubernetes
resource "yandex_kubernetes_cluster" "autolabs" {
  name        = "autolabs-cluster"
  network_id  = yandex_vpc_network.autolabs.id

  master {
    version = "1.28"
    zonal {
      zone      = "ru-central1-a"
      subnet_id = yandex_vpc_subnet.autolabs_a.id
    }

    public_ip = true
  }

  service_account_id      = yandex_iam_service_account.k8s_sa.id
  node_service_account_id = yandex_iam_service_account.k8s_nodes_sa.id
}

resource "yandex_kubernetes_node_group" "autolabs_nodes" {
  cluster_id = yandex_kubernetes_cluster.autolabs.id
  name       = "autolabs-nodes"

  instance_template {
    platform_id = "standard-v2"
    resources {
      cores  = 2
      memory = 4
    }

    boot_disk {
      type = "network-ssd"
      size = 64
    }
  }

  scale_policy {
    auto_scale {
      min     = 1
      max     = 5
      initial = 2
    }
  }
}
```

---

### Redis и Minio: Self-hosted vs Managed

#### Redis

| Вариант | Стоимость/мес | Управление | HA | Вывод |
|---------|---------------|------------|-------|-------|
| **Self-hosted Redis** (в K8s или на database-vm) | $0 | Полное | ❌ (single instance) | ✅ MVP |
| **AWS ElastiCache** (cache.t3.micro) | ~$15 | Минимальное | ✅ | ⚠️ Production |
| **Yandex Managed Redis** (hm1.nano) | ~$10 | Минимальное | ✅ | ✅ Дешевле AWS |
| **DigitalOcean Managed Redis** (db-s-1vcpu-1gb) | ~$15 | Минимальное | ❌ (за доплату) | ⚠️ Небольшие проекты |

#### Minio (S3-compatible storage)

| Вариант | Стоимость/мес | Управление | Вывод |
|---------|---------------|------------|-------|
| **Self-hosted Minio** (в K8s) | $0 | Полное | ✅ MVP |
| **AWS S3** (50 GB storage + transfer) | ~$1-5 | Нулевое | ✅ Production (очень дешево) |
| **Yandex Object Storage** (50 GB) | ~$1-3 | Нулевое | ✅ Production (дешевле AWS для РФ) |
| **Backblaze B2** (50 GB) | ~$0.3-1 | Нулевое | ✅ Самый дешёвый |

**Рекомендация:** Даже для MVP можно использовать AWS S3 или Yandex Object Storage вместо Minio — это освободит ресурсы и будет стоить ~$1-3/мес.

---

## Terraform поддержка облачных провайдеров

**Ключевое преимущество Terraform:** Универсальность. Можно описать инфраструктуру для любого провайдера.

### Поддерживаемые провайдеры

| Провайдер | Terraform Provider | Основные ресурсы |
|-----------|-------------------|------------------|
| **Proxmox** | `telmate/proxmox` | VM, LXC, storage |
| **AWS** | `hashicorp/aws` | EC2, RDS, EKS, S3, VPC, IAM |
| **Google Cloud** | `hashicorp/google` | GCE, Cloud SQL, GKE, GCS |
| **Azure** | `hashicorp/azurerm` | VM, Azure Database, AKS, Blob Storage |
| **Yandex Cloud** | `yandex-cloud/yandex` | Compute, Managed PostgreSQL, K8s, Object Storage |
| **DigitalOcean** | `digitalocean/digitalocean` | Droplets, Managed DB, Kubernetes, Spaces |
| **Hetzner** | `hetznercloud/hcloud` | Servers, Networks, Volumes |

### Пример: Multi-provider конфигурация

Terraform позволяет использовать **несколько провайдеров одновременно**:

```hcl
# providers.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    # On-premise infrastructure
    proxmox = {
      source  = "telmate/proxmox"
      version = "~> 2.9"
    }

    # Managed database в AWS
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Proxmox для K8s VM
provider "proxmox" {
  pm_api_url = var.proxmox_api_url
  pm_user    = var.proxmox_user
  pm_password = var.proxmox_password
}

# AWS для Managed PostgreSQL
provider "aws" {
  region = "eu-west-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}
```

```hcl
# k8s-vm.tf (в Proxmox)
resource "proxmox_vm_qemu" "k8s_vm" {
  name   = "k8s-vm"
  cores  = 8
  memory = 16384
  # ... остальная конфигурация
}

# database.tf (в AWS)
resource "aws_db_instance" "autolabs_postgres" {
  identifier        = "autolabs-postgres"
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  # ... остальная конфигурация
}
```

Результат: **K8s на Proxmox + PostgreSQL в AWS** — всё описано в одном Terraform проекте!

---

## Матрица принятия решения: Какой сценарий выбрать?

| Критерий | On-premise (Proxmox) | Гибридный | Полностью облачный |
|----------|---------------------|-----------|-------------------|
| **Есть физический сервер / bare-metal аренда** | ✅ Да | ⚠️ Да (но можно добавить managed сервисы) | ❌ Нет |
| **Бюджет: $0/мес** | ✅ | ❌ | ❌ |
| **Бюджет: $30-80/мес** | ✅ | ✅ | ❌ |
| **Бюджет: $150+/мес** | ✅ | ✅ | ✅ |
| **Ресурсы сервера ограничены (12 CPU, 16 GB RAM)** | ⚠️ Тесно | ✅ Managed DB освободит ресурсы | ✅ |
| **Нужна HA (99.9% uptime)** | ❌ (single-node K8s) | ✅ (managed сервисы с HA) | ✅ |
| **Учебный проект** | ✅ Идеально | ⚠️ Если есть бюджет | ❌ Дорого |
| **Production проект** | ⚠️ Нужен multi-node K8s | ✅ Оптимально | ✅ Максимальная надёжность |
| **Полный контроль над инфраструктурой** | ✅ | ⚠️ Частично (managed сервисы ограничены) | ❌ |
| **Минимальное управление** | ❌ | ⚠️ Частично | ✅ |

---

## Рекомендации по выбору

### Для учебного проекта (AutoLabs MVP):

```
✅ Сценарий A: On-premise (Proxmox)

Почему:
• Нулевая стоимость (кроме электричества/аренды сервера)
• Полный контроль для обучения студентов
• Terraform + Proxmox — хороший пример IaC
• Достаточно для 50 студентов

Альтернатива при ограниченных ресурсах:
⚠️ Сценарий B (Вариант B2): K8s + GitLab на Proxmox, только PostgreSQL в облаке
• Yandex Managed PostgreSQL (~$25/мес)
• Освобождает 4 CPU и 8 GB RAM на сервере
```

### Для production проекта с ограниченным бюджетом:

```
✅ Сценарий B (Вариант B1): Гибридный подход

On-premise (Proxmox или Hetzner Dedicated):
• Multi-node Kubernetes (3 masters + 2 workers)

Облако (Yandex Cloud — дешевле для РФ):
• Managed PostgreSQL (~$25/мес)
• Managed Redis (~$10/мес)
• Object Storage (~$2/мес)
• GitLab.com Free Tier ($0)

Итого: ~$40/мес + стоимость сервера
```

### Для production проекта с хорошим бюджетом:

```
✅ Сценарий C: Полностью облачный (AWS или Yandex Cloud)

• Managed Kubernetes (EKS / Yandex K8s)
• Managed PostgreSQL с Multi-AZ
• Managed Redis
• S3 Storage
• CloudWatch / Yandex Monitoring
• Автоматические бэкапы и DR

Итого: ~$150-300/мес в зависимости от нагрузки
Преимущество: 99.9% uptime, автоматическое масштабирование, минимум управления
```

---

## Архитектура инфраструктуры

### Схема физического и виртуального уровня

```
┌─────────────────────────────────────────────────────────────────┐
│                    Физический сервер                             │
│                  Proxmox VE (Hypervisor)                         │
│  • CPU: 12-24 cores                                              │
│  • RAM: 16-32 GB                                                 │
│  • Disk: 500 GB - 1 TB SSD                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Виртуальная сеть: vmbr0 (Bridge)                               │
│  ├─ 192.168.100.0/24 (единая подсеть для MVP)                   │
│  │                                                                │
│  ├─ 192.168.100.10 → gitlab-vm                                   │
│  ├─ 192.168.100.20 → database-vm                                 │
│  └─ 192.168.100.30 → k8s-vm                                      │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────┐                                        │
│  │   gitlab-vm          │                                        │
│  ├──────────────────────┤                                        │
│  │ • OS: Ubuntu 22.04   │                                        │
│  │ • CPU: 4 cores       │                                        │
│  │ • RAM: 8 GB          │                                        │
│  │ • Disk: 100 GB       │                                        │
│  │ • IP: 192.168.100.10 │                                        │
│  │                      │                                        │
│  │ Установлено:         │                                        │
│  │ • GitLab CE          │                                        │
│  │ • GitLab Runner      │                                        │
│  │ • Container Registry │                                        │
│  └──────────────────────┘                                        │
│                                                                   │
│  ┌──────────────────────┐                                        │
│  │   database-vm        │                                        │
│  ├──────────────────────┤                                        │
│  │ • OS: Ubuntu 22.04   │                                        │
│  │ • CPU: 4 cores       │                                        │
│  │ • RAM: 8 GB          │                                        │
│  │ • Disk: 200 GB       │                                        │
│  │ • IP: 192.168.100.20 │                                        │
│  │                      │                                        │
│  │ Установлено:         │                                        │
│  │ • PostgreSQL 15      │                                        │
│  │   ├─ DB: autolabs    │                                        │
│  │   ├─ DB: zitadel     │                                        │
│  │   └─ DB: gitlab      │                                        │
│  │ • Redis 7            │                                        │
│  │ • Minio              │                                        │
│  └──────────────────────┘                                        │
│                                                                   │
│  ┌──────────────────────┐                                        │
│  │   k8s-vm             │                                        │
│  ├──────────────────────┤                                        │
│  │ • OS: Ubuntu 22.04   │                                        │
│  │ • CPU: 4-16 cores    │                                        │
│  │ • RAM: 8-16 GB       │                                        │
│  │ • Disk: 200 GB       │                                        │
│  │ • IP: 192.168.100.30 │                                        │
│  │                      │                                        │
│  │ Установлено:         │                                        │
│  │ • Kubernetes 1.28+   │                                        │
│  │ • containerd         │                                        │
│  │ • CNI: Calico        │                                        │
│  │ • Ingress: nginx     │                                        │
│  │                      │                                        │
│  │ Namespaces:          │                                        │
│  │ • platform           │                                        │
│  │ • infrastructure     │                                        │
│  │ • auth               │                                        │
│  │ • messaging          │                                        │
│  │ • labs               │                                        │
│  │ • ingress-nginx      │                                        │
│  └──────────────────────┘                                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Сетевая архитектура

### Вариант A: Единая сеть (MVP) ✅

**Выбрано для начального этапа**

```
Все VM находятся в одной подсети:
192.168.100.0/24

Преимущества:
✅ Простая настройка
✅ Все компоненты видят друг друга
✅ Не требуется настройка routing/firewall

Недостатки:
⚠️ Нет изоляции между компонентами
⚠️ Все сервисы в одном broadcast domain

Безопасность обеспечивается:
• Firewall на уровне каждой VM (ufw)
• Network Policies в Kubernetes
• Отсутствие публичного доступа к внутренним сервисам
```

### Вариант B: Сегментированная сеть (Future)

```
Для production рекомендуется разделить на VLAN'ы:

VLAN 10: Management Network (192.168.10.0/24)
├─ Proxmox Web UI
└─ SSH доступ к VM

VLAN 100: Infrastructure (192.168.100.0/24)
├─ 192.168.100.10 → gitlab-vm
├─ 192.168.100.20 → database-vm
└─ 192.168.100.30 → k8s-vm (control plane)

VLAN 200: Kubernetes Workloads (192.168.200.0/24)
└─ Pod network (calico)

VLAN 300: Labs Network (192.168.300.0/24)
└─ Изолированная сеть для лабораторных работ студентов

DMZ (10.0.0.0/24)
└─ Публичный доступ через Ingress
```

**Для MVP используем Вариант A.** Переход на Вариант B возможен позже без переделки инфраструктуры.

---

## Конфигурация PostgreSQL

### Вариант A: Единый инстанс PostgreSQL ✅

**Выбрано для проекта**

```sql
-- Единый PostgreSQL сервер с тремя базами данных

CREATE DATABASE autolabs;   -- Основная БД платформы
CREATE DATABASE zitadel;    -- БД для Zitadel (авторизация)
CREATE DATABASE gitlab;     -- БД для GitLab CE

-- Отдельные пользователи для каждой БД
CREATE USER autolabs_user WITH PASSWORD 'strong_password';
CREATE USER zitadel_user WITH PASSWORD 'strong_password';
CREATE USER gitlab_user WITH PASSWORD 'strong_password';

-- Выдача прав
GRANT ALL PRIVILEGES ON DATABASE autolabs TO autolabs_user;
GRANT ALL PRIVILEGES ON DATABASE zitadel TO zitadel_user;
GRANT ALL PRIVILEGES ON DATABASE gitlab TO gitlab_user;
```

**Преимущества:**
- ✅ Простое управление (один сервер)
- ✅ Экономия ресурсов (один процесс PostgreSQL)
- ✅ Единый бэкап всех баз данных
- ✅ Упрощённый мониторинг

**Недостатки:**
- ⚠️ Single Point of Failure (падение PostgreSQL → падение всей платформы)
- ⚠️ Нельзя масштабировать отдельные базы

**Митигация рисков:**
- Регулярные snapshot'ы Proxmox для database-vm
- Настройка WAL архивирования PostgreSQL
- Для production: переход на PostgreSQL кластер (Patroni)

### Вариант B: Отдельные PostgreSQL инстансы (Future)

```
Для высоконагруженных систем:

• PostgreSQL для autolabs (отдельная VM)
• PostgreSQL для zitadel (отдельная VM)
• PostgreSQL для gitlab (отдельная VM)

Или использование управляемых БД:
• AWS RDS PostgreSQL
• Google Cloud SQL
• Managed PostgreSQL от Hetzner
```

---

## Kubernetes: Single-node vs Multi-node

### Вариант A: Single-node кластер ✅

**Выбрано для MVP**

```
k8s-vm (192.168.100.30)
└─ Один узел выполняет роли:
   • Control Plane (kube-apiserver, etcd, scheduler)
   • Worker Node (kubelet, container runtime)

Установка: kubeadm + containerd

# Разрешаем запуск pod'ов на control plane node
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Преимущества:**
- ✅ Простая установка и управление
- ✅ Минимальные требования к ресурсам
- ✅ Подходит для учебного проекта и MVP

**Недостатки:**
- ⚠️ Нет отказоустойчивости (один узел → SPOF)
- ⚠️ Обновления требуют downtime

**Когда использовать:**
- MVP и proof of concept
- Учебные и dev окружения
- Проекты с небольшой нагрузкой

### Вариант B: Multi-node кластер (Future)

```
Для production:

k8s-master-1 (192.168.100.31) → Control Plane
k8s-master-2 (192.168.100.32) → Control Plane
k8s-master-3 (192.168.100.33) → Control Plane (etcd требует 3+ узла)

k8s-worker-1 (192.168.100.41) → Worker Node
k8s-worker-2 (192.168.100.42) → Worker Node
k8s-worker-3 (192.168.100.43) → Worker Node

Установка: kubeadm + HA setup + внешний load balancer
```

**Преимущества:**
- ✅ Высокая доступность (HA)
- ✅ Горизонтальное масштабирование
- ✅ Обновления без downtime

**Недостатки:**
- ⚠️ Требует минимум 3 control plane nodes + 2 worker nodes = 5 VM
- ⚠️ Сложнее в настройке и поддержке

---

## Terraform конфигурация

### Структура проекта

```
terraform/
├── main.tf               # Главный файл конфигурации
├── variables.tf          # Переменные
├── outputs.tf            # Выходные данные
├── providers.tf          # Провайдеры (Proxmox)
├── gitlab-vm.tf          # Конфигурация GitLab VM
├── database-vm.tf        # Конфигурация Database VM
├── k8s-vm.tf             # Конфигурация Kubernetes VM
├── network.tf            # Настройки сети
└── terraform.tfvars      # Значения переменных (НЕ коммитить!)
```

### Пример: providers.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "~> 2.9"
    }
  }
}

provider "proxmox" {
  pm_api_url      = var.proxmox_api_url
  pm_user         = var.proxmox_user
  pm_password     = var.proxmox_password
  pm_tls_insecure = true  # Для self-signed сертификатов
}
```

### Пример: variables.tf

```hcl
variable "proxmox_api_url" {
  description = "URL Proxmox API"
  type        = string
  default     = "https://192.168.1.100:8006/api2/json"
}

variable "proxmox_user" {
  description = "Proxmox пользователь"
  type        = string
  default     = "root@pam"
}

variable "proxmox_password" {
  description = "Пароль Proxmox"
  type        = string
  sensitive   = true
}

variable "ssh_public_key" {
  description = "SSH публичный ключ для доступа к VM"
  type        = string
}

variable "vm_template" {
  description = "ID template Ubuntu 22.04 в Proxmox"
  type        = string
  default     = "9000"
}
```

### Пример: gitlab-vm.tf

```hcl
resource "proxmox_vm_qemu" "gitlab_vm" {
  name        = "gitlab-vm"
  desc        = "GitLab CE + Container Registry + Runner"
  target_node = var.proxmox_node

  # Клонирование из template
  clone = var.vm_template

  # Ресурсы
  cores   = 4
  sockets = 1
  memory  = 8192  # 8 GB

  # Диск
  disk {
    size    = "100G"
    type    = "scsi"
    storage = "local-lvm"
    format  = "raw"
  }

  # Сеть
  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  # IP адрес
  ipconfig0 = "ip=192.168.100.10/24,gw=192.168.100.1"

  # SSH ключ
  sshkeys = var.ssh_public_key

  # Cloud-init
  os_type   = "cloud-init"
  ciuser    = "ubuntu"
  cipassword = var.vm_password

  # Автозапуск при старте Proxmox
  onboot = true
}
```

### Пример: database-vm.tf

```hcl
resource "proxmox_vm_qemu" "database_vm" {
  name        = "database-vm"
  desc        = "PostgreSQL + Redis + Minio"
  target_node = var.proxmox_node

  clone = var.vm_template

  cores   = 4
  sockets = 1
  memory  = 8192

  disk {
    size    = "200G"
    type    = "scsi"
    storage = "local-lvm"
    format  = "raw"
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  ipconfig0 = "ip=192.168.100.20/24,gw=192.168.100.1"

  sshkeys = var.ssh_public_key

  os_type   = "cloud-init"
  ciuser    = "ubuntu"
  cipassword = var.vm_password

  onboot = true
}
```

### Пример: k8s-vm.tf

```hcl
resource "proxmox_vm_qemu" "k8s_vm" {
  name        = "k8s-vm"
  desc        = "Single-node Kubernetes cluster"
  target_node = var.proxmox_node

  clone = var.vm_template

  # Гибкие ресурсы в зависимости от окружения
  cores   = var.k8s_cores    # 4 для dev, 16 для prod
  sockets = 1
  memory  = var.k8s_memory   # 8192 для dev, 16384 для prod

  disk {
    size    = "200G"
    type    = "scsi"
    storage = "local-lvm"
    format  = "raw"
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
  }

  ipconfig0 = "ip=192.168.100.30/24,gw=192.168.100.1"

  sshkeys = var.ssh_public_key

  os_type   = "cloud-init"
  ciuser    = "ubuntu"
  cipassword = var.vm_password

  onboot = true
}
```

### Применение конфигурации

```bash
# 1. Инициализация Terraform
cd terraform/
terraform init

# 2. Проверка плана (какие ресурсы будут созданы)
terraform plan

# 3. Применение изменений
terraform apply

# 4. Вывод информации (IP адреса VM)
terraform output

# 5. Удаление всех ресурсов (при необходимости)
terraform destroy
```

**Время создания инфраструктуры:** 5-10 минут

---

## Создание template Ubuntu 22.04 в Proxmox

Перед запуском Terraform необходимо создать **cloud-init template** в Proxmox:

```bash
# Подключиться к Proxmox по SSH
ssh root@proxmox-server

# Скачать Ubuntu 22.04 Cloud Image
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Создать VM template
qm create 9000 --name ubuntu-22.04-template --memory 2048 --net0 virtio,bridge=vmbr0
qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Конвертировать VM в template
qm template 9000

echo "Template создан с ID: 9000"
```

Теперь Terraform будет клонировать VM из этого template.

---

## Резервное копирование

### Стратегия A: Proxmox Snapshots ✅

**Выбрано для проекта**

Proxmox VE имеет встроенную систему snapshot'ов:

```bash
# Создание snapshot'а для gitlab-vm
qm snapshot <VM_ID> snap-$(date +%Y%m%d-%H%M%S) --description "GitLab backup"

# Пример через UI:
Proxmox UI → VM → Snapshots → Take Snapshot
```

**Конфигурация автоматических бэкапов:**

```bash
# В Proxmox Web UI:
Datacenter → Backup → Add

Настройки:
• Schedule: Daily 02:00
• Storage: local (или NFS/SMB share)
• Mode: Snapshot
• Compression: ZSTD
• Retention: Keep last 7 daily, 4 weekly
```

**Преимущества:**
- ✅ Встроено в Proxmox
- ✅ Быстрое создание snapshot'а (секунды)
- ✅ Откат на предыдущее состояние за минуты
- ✅ Не требует дополнительных инструментов

**Недостатки:**
- ⚠️ Snapshot'ы хранятся на том же сервере (если сервер сгорит → потеря данных)
- ⚠️ Занимают место на диске

**Митигация:** Настроить Proxmox Backup Server (PBS) на отдельном сервере/NAS для off-site бэкапов.

### Стратегия B: Application-level backups

Дополнительно к snapshot'ам VM рекомендуется делать логические бэкапы:

```bash
# PostgreSQL dump (внутри database-vm)
pg_dumpall -U postgres > /backup/postgres-$(date +%Y%m%d).sql

# GitLab backup (внутри gitlab-vm)
gitlab-backup create

# Kubernetes etcd backup (внутри k8s-vm)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
```

---

## Мониторинг инфраструктуры

Для мониторинга состояния VM и Proxmox рекомендуется (в будущем):

```
• Prometheus + Grafana → Метрики VM (CPU, RAM, Disk I/O)
• Proxmox VE Exporter → Экспорт метрик в Prometheus
• Alertmanager → Уведомления в Telegram при проблемах

Метрики:
- Загрузка CPU > 80%
- Использование RAM > 90%
- Свободное место на диске < 20%
- Время отклика VM > 5 секунд
```

---

## Этапы развёртывания инфраструктуры

### Этап 1: Подготовка Proxmox

```bash
1. Установить Proxmox VE 8.x на физический сервер
2. Настроить сеть (vmbr0 bridge)
3. Создать cloud-init template Ubuntu 22.04 (ID: 9000)
4. Создать API токен для Terraform
```

### Этап 2: Развёртывание VM через Terraform

```bash
1. Клонировать репозиторий с Terraform конфигурацией
2. Заполнить terraform.tfvars с credentials
3. Выполнить terraform init && terraform apply
4. Дождаться создания 3 VM (gitlab-vm, database-vm, k8s-vm)
```

### Этап 3: Настройка GitLab VM

```bash
1. SSH в gitlab-vm: ssh ubuntu@192.168.100.10
2. Установить Docker Engine
3. Развернуть GitLab CE через Docker Compose
4. Настроить GitLab Runner
5. Включить Container Registry
```

### Этап 4: Настройка Database VM

```bash
1. SSH в database-vm: ssh ubuntu@192.168.100.20
2. Установить PostgreSQL 15
3. Создать три базы данных: autolabs, zitadel, gitlab
4. Установить Redis 7
5. Установить Minio (S3-compatible storage)
```

### Этап 5: Настройка Kubernetes VM

```bash
1. SSH в k8s-vm: ssh ubuntu@192.168.100.30
2. Установить containerd
3. Установить kubeadm, kubelet, kubectl
4. Инициализировать single-node кластер
5. Установить Calico CNI
6. Установить Ingress NGINX Controller
7. Создать namespaces: platform, infrastructure, auth, messaging, labs
```

**Общее время развёртывания:** 2-4 часа при ручной настройке, 30-60 минут с автоматизацией.

---

## Масштабирование (Roadmap)

### Фаза 1: MVP (текущая)

```
Single-node всё:
• 1 Proxmox сервер
• 3 VM (gitlab, database, k8s)
• Single-node Kubernetes
• Единая сеть
• Один PostgreSQL инстанс

Поддерживает: до 50 студентов, 5-10 лабораторных работ одновременно
```

### Фаза 2: Production-ready

```
Добавить отказоустойчивость:
• Multi-node Kubernetes (3 control plane + 2 workers)
• PostgreSQL кластер (Patroni + etcd)
• GitLab HA (несколько runner'ов)
• Сегментированная сеть (VLAN'ы)
• Proxmox Backup Server для off-site бэкапов

Поддерживает: до 200 студентов, 30-50 лабораторных работ одновременно
```

### Фаза 3: Enterprise

```
Масштабирование на несколько серверов:
• Proxmox кластер (3+ узла)
• Ceph/GlusterFS для shared storage
• Kubernetes multi-cluster (ArgoCD для GitOps)
• PostgreSQL с read replicas
• CDN для статики

Поддерживает: 500+ студентов, 100+ одновременных лабораторных работ
```

---

## Вывод

Инфраструктура AutoLabs построена на принципах:

1. **Воспроизводимость:** Terraform позволяет создать инфраструктуру за 10 минут (любой вариант: on-premise, гибрид, облако)
2. **Гибкость выбора:** Три сценария развёртывания — от $0/мес (on-premise) до $150-300/мес (полностью облачный)
3. **Простота (для MVP):** Single-node Kubernetes, единая сеть, один PostgreSQL — минимальная сложность
4. **Масштабируемость:** Лёгкий переход от MVP к production через гибридный подход (managed сервисы)
5. **Контроль vs Удобство:** On-premise = полный контроль, Облако = минимум управления
6. **Современность:** IaC (Terraform), контейнеризация (K8s), CI/CD (GitLab)

**Критически важно:**

1. **Terraform — обязательное требование** для всех сценариев развёртывания. Это позволяет:
   - Разработчикам быстро развернуть копию платформы
   - Преподавателям продемонстрировать современный подход к IaC
   - Легко переключаться между on-premise, гибридным и облачным вариантами

2. **Выбор сценария зависит от ресурсов:**
   - **Есть сервер + $0 бюджет** → On-premise (Proxmox)
   - **Ограниченные ресурсы сервера** → Гибрид (вынести PostgreSQL в облако = -4 CPU, -8 GB RAM)
   - **Нет сервера** → Полностью облачный (AWS, GCP, Yandex Cloud)

3. **Terraform поддерживает multi-provider конфигурации:**
   - Можно комбинировать Proxmox (K8s VM) + AWS (RDS) + Yandex Cloud (Object Storage) в одном проекте
   - Единая точка управления инфраструктурой через `.tf` файлы
   - Версионирование в Git для всех вариантов развёртывания

**Рекомендация для AutoLabs MVP:** On-premise (Proxmox) с возможностью миграции на гибридный подход при росте нагрузки.

---

## Дополнительные ресурсы

**Документация:**
- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/)
- [Terraform Proxmox Provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

**Следующие этапы документации:**
- `02-backend-platform/` — Backend Platform (FastAPI, бизнес-логика)
- `03-celery-workers/` — Celery Workers (K8s API, инфраструктурные операции)
- `05-kubernetes/` — Kubernetes (namespaces, deployments, services)
- `07-lab-deployments/` — Деплой лабораторных работ студентов
- `08-network-flows/` — Потоки данных между компонентами
