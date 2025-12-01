# Система авторизации AutoLabs (Zitadel)

## Дата создания
2025-12-01

---

## Обзор

**AutoLabs** использует централизованную систему авторизации на базе **Zitadel** - современного open-source решения для Identity and Access Management (IAM).

### Ключевые принципы:

1. **Ролевая модель (RBAC)** - три базовые роли: Admin, Teacher, Student
2. **Атрибутная модель (ABAC)** - проверка владения ресурсами (курсами, студентами)
3. **OAuth2/OIDC** - стандартный протокол авторизации
4. **JWT токены** - безопасная передача информации о пользователе
5. **Централизованное управление** - единая точка авторизации для всех компонентов

---

## Архитектура авторизации

```
┌─────────────┐
│   Frontend  │
└──────┬──────┘
       │ 1. Redirect to login
       ↓
┌─────────────┐
│   Zitadel   │ ← OAuth2/OIDC Provider
│  (namespace:│
│    auth)    │
└──────┬──────┘
       │ 2. JWT Token
       ↓
┌─────────────┐
│   Frontend  │
└──────┬──────┘
       │ 3. API request + JWT
       ↓
┌──────────────────┐
│ Backend Platform │
│                  │
│ 4. Validate JWT  │ ← Проверка подписи, срока
│ 5. Check Role    │ ← Проверка роли (admin/teacher/student)
│ 6. Check Attrs   │ ← Проверка атрибутов (owner_of_course, etc)
└──────┬───────────┘
       │ 7. Allow / Deny
       ↓
    Response
```

---

## Три базовые роли

### 1. **Admin (Администратор)**
**Назначение:** Полное управление платформой

**Может:**
- ✅ Управление пользователями (создание, удаление, блокировка)
- ✅ Управление ВСЕМИ курсами (любого преподавателя)
- ✅ Просмотр всех лабораторных работ
- ✅ Загрузка и настройка лабораторных работ
- ✅ Настройка системы (квоты ресурсов, глобальные параметры)
- ✅ Доступ к мониторингу и логам
- ✅ Добавление новых учебных модулей

**Не может:**
- Нет ограничений

---

### 2. **Teacher (Преподаватель)**
**Назначение:** Управление образовательным процессом в рамках своих курсов

**Может:**
- ✅ Создание/редактирование/удаление СВОИХ курсов
- ✅ Запуск/остановка лабораторных работ для групп СВОИХ студентов
- ✅ Просмотр прогресса СВОИХ студентов
- ✅ Оценивание СВОИХ студентов
- ✅ Управление группами студентов в СВОИХ курсах
- ✅ (Опционально) Создание заявок на загрузку новых лабораторных работ

**Не может:**
- ❌ Загружать конфигурации лабораторных работ напрямую
- ❌ Видеть курсы других преподавателей
- ❌ Добавлять студентов в курсы (только Admin)
- ❌ Видеть прогресс студентов других преподавателей
- ❌ Изменять системные настройки
- ❌ Доступ к мониторингу и логам инфраструктуры

---

### 3. **Student (Студент)**
**Назначение:** Выполнение лабораторных работ и отслеживание своего прогресса

**Может:**
- ✅ Просмотр назначенных курсов
- ✅ Доступ к SSH лабораторных работ (когда преподаватель запустил)
- ✅ Просмотр своего прогресса
- ✅ Отправка результатов выполнения заданий
- ✅ Просмотр своих оценок

**Не может:**
- ❌ Создавать курсы
- ❌ Видеть прогресс других студентов
- ❌ Запускать/останавливать лабораторные работы
- ❌ Изменять конфигурации
- ❌ Видеть курсы, к которым не назначен

---

## Атрибутная модель доступа (ABAC)

### Концепция

**Роль** определяет базовые права, но **атрибуты** ограничивают доступ к конкретным ресурсам.

### Основные атрибуты:

| Атрибут | Описание | Пример |
|---------|----------|--------|
| `owner_of_course(course_id)` | Преподаватель - владелец курса | Teacher может редактировать только свой курс |
| `assigned_to_course(course_id)` | Студент назначен на курс | Student видит только свои курсы |
| `member_of_group(group_id)` | Студент член группы | Student получает доступ к лабам своей группы |
| `teacher_of_student(student_id)` | Преподаватель - учитель студента | Teacher видит прогресс только своих студентов |

### Примеры проверок:

#### Пример 1: Преподаватель запускает лабу

```
Запрос: POST /api/v1/labs/123/start
JWT токен: teacher@example.com (роль: Teacher)
Ресурс: Лаба 123 в курсе "Кибербезопасность-101"

Проверка Backend Platform:
1. Роль = Teacher? ✅
2. owner_of_course("Кибербезопасность-101")? ✅ (проверка в PostgreSQL)
3. Действие "start_lab" разрешено для Teacher? ✅

Результат: 200 OK, лаба запускается
```

#### Пример 2: Преподаватель пытается запустить чужую лабу

```
Запрос: POST /api/v1/labs/456/start
JWT токен: teacher@example.com (роль: Teacher)
Ресурс: Лаба 456 в курсе "Сети-202" (курс другого преподавателя)

Проверка Backend Platform:
1. Роль = Teacher? ✅
2. owner_of_course("Сети-202")? ❌ (НЕ владелец)
3. Действие "start_lab" для чужого курса? ❌

Результат: 403 Forbidden
{
  "error": "Access denied",
  "message": "You are not the owner of this course"
}
```

#### Пример 3: Студент получает доступ к лабе

```
Запрос: GET /api/v1/labs/123/access
JWT токен: student@example.com (роль: Student)
Ресурс: SSH credentials для лабы 123

Проверка Backend Platform:
1. Роль = Student? ✅
2. assigned_to_course(course_id лабы 123)? ✅ (студент в курсе)
3. member_of_group(group_id лабы)? ✅ (лаба запущена для его группы)
4. Лаба в статусе "running"? ✅

Результат: 200 OK
{
  "ssh_host": "labs.autolabs.local",
  "ssh_port": 30123,
  "username": "student123",
  "password": "generated_password"
}
```

---

## Процесс загрузки лабораторных работ

### Проблема
Преподаватели не могут загружать конфигурации лабораторных работ напрямую по соображениям безопасности (риск вредоносного кода, уязвимостей).

### Решение: Два варианта

#### Вариант A: Прямое обращение к Admin

```
1. Teacher → Обращается к Admin (вне системы: email, чат)
2. Admin → Проверяет конфигурацию лабы (безопасность, корректность)
3. Admin → Загружает лабу через GitLab (push в autolabs/labs/)
4. GitLab CI → Билдит образ, пушит в Registry
5. Admin → Добавляет лабу в каталог платформы (через админ-панель)
6. Teacher → Видит новую лабу в списке доступных, может запускать
```

#### Вариант B: Система заявок (рекомендуется для масштабирования)

```
1. Teacher → Создаёт заявку на загрузку лабы (POST /api/v1/labs/requests)
   - Загружает конфигурацию (YAML)
   - Загружает Dockerfile(s)
   - Описание лабораторной работы

2. Backend Platform → Сохраняет заявку (статус: "pending")

3. Admin → Получает уведомление о новой заявке

4. Admin → Проверяет заявку через админ-панель:
   - Просмотр конфигурации
   - Сканирование на безопасность (опционально: автоматический Trivy scan)
   - Проверка корректности

5. Admin → Принимает решение:

   A) Одобрить (POST /api/v1/labs/requests/123/approve):
      - Статус заявки: "approved"
      - Автоматический push в GitLab (autolabs/labs/)
      - GitLab CI билдит образ
      - Лаба добавляется в каталог
      - Teacher получает уведомление

   B) Отклонить (POST /api/v1/labs/requests/123/reject):
      - Статус заявки: "rejected"
      - Admin указывает причину отклонения
      - Teacher получает уведомление с комментариями

6. Teacher → Видит статус заявки, может использовать одобренную лабу
```

**Рекомендация:** Вариант B для production, Вариант A для MVP.

---

## OAuth2/OIDC Flow

### Процесс логина (Authorization Code Flow)

```
1. User → Frontend → Клик "Войти"

2. Frontend → Redirect на Zitadel:
   GET https://auth.autolabs.local/oauth2/authorize?
       client_id=autolabs-frontend
       &redirect_uri=https://autolabs.local/callback
       &response_type=code
       &scope=openid profile email

3. Zitadel → Показывает форму логина

4. User → Вводит логин/пароль

5. Zitadel → Проверяет credentials

6. Zitadel → Redirect обратно на Frontend:
   https://autolabs.local/callback?code=AUTH_CODE_123

7. Frontend → Обменивает code на токены (backend-for-frontend):
   POST https://auth.autolabs.local/oauth2/token
   Body:
     grant_type=authorization_code
     code=AUTH_CODE_123
     client_id=autolabs-frontend
     client_secret=SECRET
     redirect_uri=https://autolabs.local/callback

8. Zitadel → Возвращает токены:
   {
     "access_token": "eyJhbGciOiJSUzI1NiIs...",
     "id_token": "eyJhbGciOiJSUzI1NiIs...",
     "refresh_token": "...",
     "expires_in": 3600,
     "token_type": "Bearer"
   }

9. Frontend → Сохраняет access_token (memory, не localStorage)

10. Frontend → Все API запросы с токеном:
    GET /api/v1/courses
    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

---

## Структура JWT токена

### ID Token (для информации о пользователе)

```json
{
  "iss": "https://auth.autolabs.local",
  "sub": "user-id-123",
  "aud": "autolabs-frontend",
  "exp": 1701445200,
  "iat": 1701441600,
  "email": "teacher@example.com",
  "email_verified": true,
  "name": "Иван Иванов",
  "preferred_username": "i.ivanov",
  "roles": ["teacher"],
  "org": {
    "organization_id": "org-123",
    "organization_name": "МИФИ"
  }
}
```

### Access Token (для авторизации API запросов)

```json
{
  "iss": "https://auth.autolabs.local",
  "sub": "user-id-123",
  "aud": "autolabs-backend",
  "exp": 1701445200,
  "iat": 1701441600,
  "scope": "openid profile email courses:read courses:write",
  "roles": ["teacher"],
  "permissions": [
    "course.create",
    "course.update",
    "course.delete",
    "lab.start",
    "lab.stop",
    "student.view_progress"
  ]
}
```

---

## Валидация токена в Backend Platform

### Процесс проверки

```python
# backend-platform/auth/jwt_validator.py

from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthBearer
from jose import jwt, JWTError
import httpx

security = HTTPBearer()

async def validate_jwt(token: str):
    """
    Валидация JWT токена
    """
    try:
        # 1. Получение публичного ключа от Zitadel (кэшируется)
        jwks = await get_jwks_from_zitadel()

        # 2. Декодирование и проверка подписи
        payload = jwt.decode(
            token,
            jwks,
            algorithms=["RS256"],
            audience="autolabs-backend",
            issuer="https://auth.autolabs.local"
        )

        # 3. Проверка срока действия (автоматически в jwt.decode)

        # 4. Извлечение информации о пользователе
        user_id = payload.get("sub")
        roles = payload.get("roles", [])
        permissions = payload.get("permissions", [])

        return {
            "user_id": user_id,
            "roles": roles,
            "permissions": permissions,
            "email": payload.get("email")
        }

    except JWTError as e:
        raise HTTPException(status_code=401, detail="Invalid token")

# Использование в API endpoints
@app.get("/api/v1/courses")
async def get_courses(token: str = Security(security)):
    user = await validate_jwt(token.credentials)

    # Проверка роли
    if "teacher" in user["roles"]:
        # Вернуть только курсы этого преподавателя
        courses = await get_courses_by_teacher(user["user_id"])
    elif "admin" in user["roles"]:
        # Вернуть все курсы
        courses = await get_all_courses()
    elif "student" in user["roles"]:
        # Вернуть только назначенные курсы
        courses = await get_assigned_courses(user["user_id"])
    else:
        raise HTTPException(status_code=403, detail="Access denied")

    return courses
```

---

## Проверка атрибутов (ABAC)

### Пример: Проверка владения курсом

```python
# backend-platform/auth/abac.py

async def check_course_owner(user_id: str, course_id: str) -> bool:
    """
    Проверка, является ли пользователь владельцем курса
    """
    query = """
        SELECT COUNT(*)
        FROM courses
        WHERE id = :course_id AND teacher_id = :user_id
    """
    result = await db.fetch_one(query, {
        "course_id": course_id,
        "user_id": user_id
    })
    return result[0] > 0

# Использование в API endpoint
@app.post("/api/v1/labs/{lab_id}/start")
async def start_lab(
    lab_id: str,
    token: str = Security(security)
):
    user = await validate_jwt(token.credentials)

    # 1. Проверка роли
    if "teacher" not in user["roles"]:
        raise HTTPException(status_code=403, detail="Only teachers can start labs")

    # 2. Получение курса лабы
    lab = await get_lab(lab_id)
    course_id = lab.course_id

    # 3. Проверка владения курсом (ABAC)
    is_owner = await check_course_owner(user["user_id"], course_id)
    if not is_owner:
        raise HTTPException(
            status_code=403,
            detail="You are not the owner of this course"
        )

    # 4. Запуск лабы
    await start_lab_deployment(lab_id)

    return {"status": "started"}
```

---

## Бизнес-сценарии

### Сценарий 1: Преподаватель создаёт курс

```
1. Teacher логинится → Получает JWT с role=teacher
2. Frontend → POST /api/v1/courses
   Body: {"name": "Кибербезопасность-101", "description": "..."}
   Header: Authorization: Bearer <token>
3. Backend Platform:
   - Валидация JWT ✅
   - Проверка роли: "teacher" in roles ✅
   - Проверка пермишана: "course.create" in permissions ✅
   - Создание курса в PostgreSQL (owner_id = teacher_id)
4. Response: 201 Created, {"id": "course-123", "name": "..."}
```

### Сценарий 2: Admin назначает студентов на курс

```
1. Admin логинится → JWT с role=admin
2. Frontend → POST /api/v1/courses/123/students
   Body: {"student_ids": ["student-1", "student-2", "student-3"]}
3. Backend Platform:
   - Валидация JWT ✅
   - Проверка роли: "admin" in roles ✅
   - Добавление студентов в курс (таблица course_enrollments)
4. Response: 200 OK
5. Студенты получают уведомления о назначении
```

### Сценарий 3: Студент получает доступ к лабе

```
1. Teacher запускает лабу для группы (см. Сценарий 1)
2. Celery Worker создаёт поды, отправляет credentials в RabbitMQ
3. Backend Platform получает credentials, обновляет PostgreSQL
4. Student логинится → JWT с role=student
5. Frontend → GET /api/v1/labs/123/access
6. Backend Platform:
   - Валидация JWT ✅
   - Проверка роли: "student" in roles ✅
   - Проверка атрибута: assigned_to_course(course_id) ✅
   - Проверка атрибута: member_of_group(group_id) ✅
   - Проверка статуса лабы: "running" ✅
   - Возврат SSH credentials
7. Response: 200 OK, {"ssh_host": "...", "ssh_port": 30123, ...}
8. Student подключается: ssh student123@labs.autolabs.local -p 30123
```

### Сценарий 4: Преподаватель пытается увидеть чужой курс

```
1. Teacher логинится
2. Frontend → GET /api/v1/courses/456
   (курс 456 принадлежит другому преподавателю)
3. Backend Platform:
   - Валидация JWT ✅
   - Проверка роли: "teacher" in roles ✅
   - Проверка владения: owner_of_course(456) ❌
4. Response: 403 Forbidden
   {"error": "Access denied", "message": "You cannot view courses of other teachers"}
```

---

## Документация в этой секции

| Файл | Описание |
|------|----------|
| `README.md` | Общий обзор системы авторизации (этот файл) |
| `roles-permissions.md` | Детальное описание трёх ролей и их пермишанов |
| `oauth2-flow.md` | Детальное описание OAuth2/OIDC процесса |
| `attribute-based-access.md` | ABAC модель и примеры проверок |
| `lab-approval-workflow.md` | Процесс загрузки и одобрения лаб (заявки) |
| `jwt-structure.md` | Структура JWT токенов и их валидация |
| `auth-flow.drawio` | Схема потока авторизации (визуализация) |

---

## Будущие улучшения (вне MVP)

### Multi-Factor Authentication (MFA)
- TOTP (Google Authenticator, Authy)
- SMS verification
- Hardware tokens (YubiKey)

### Дополнительные роли
- **TA (Teaching Assistant)** - помощник преподавателя
- **Course Admin** - администратор курсов (не системный)
- **Lab Moderator** - модератор лабораторных работ

### Аудит
- Логирование всех действий пользователей
- Отслеживание доступа к конфиденциальным данным
- Alerts при подозрительной активности

### Session Management
- Управление активными сессиями
- Принудительный logout при смене пароля
- Ограничение concurrent sessions

---

**Версия:** 1.0
**Дата последнего обновления:** 2025-12-01
