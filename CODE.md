# DoveChat Backend — Архитектура и общие принципы

## Содержание

1. [Общий подход](#1-общий-подход)
2. [Структура монорепы](#2-структура-монорепы)
3. [База данных и подключение](#3-база-данных-и-подключение)
4. [Базовый репозиторий](#4-базовый-репозиторий)
5. [Паттерн создания сервисов](#5-паттерн-создания-сервисов)
6. [Авторизация и роли](#6-авторизация-и-роли)
7. [Общие инфраструктурные модули](#7-общие-инфраструктурные-модули)

---

## 1. Общий подход

### Domain-Driven Design (DDD)

Бэкенд построен по принципам DDD в лёгкой форме: домены разделены по сервисам (account, chat/api, admin, crm), каждый сервис инкапсулирует свою область ответственности. Общая бизнес-логика и инфраструктура вынесена в корень монорепы и переиспользуется всеми сервисами через импорт.

**Ключевые принципы:**

- **Модели** — источник истины о структуре данных, живут в `/mono/models/`, импортируются только в репозитории.
- **Репозитории** — единственный способ работы с БД. Вся работа с SQLAlchemy — только здесь.
- **Сервисы** — вся бизнес-логика. Не знают о HTTP, не знают о SQLAlchemy — работают только через репозитории.
- **Ручки (роутеры)** — тонкий слой HTTP. Только вызов сервиса и возврат результата. Никакой логики.
- **Схемы (Pydantic)** — валидация входящих данных и сериализация ответов. Живут в `/mono/schemas/` или в `schemas/` внутри сервиса.

### Запрещённые паттерны

```python
# НЕЛЬЗЯ: модели в ручках
from models.tenants import User  # в router.py — ОШИБКА

# НЕЛЬЗЯ: SQLAlchemy в сервисах
from sqlalchemy import select  # в service.py — ОШИБКА

# НЕЛЬЗЯ: бизнес-логика в ручках
@router.post("/users")
async def create_user(data: UserCreate):
    user = await db.execute(...)  # ОШИБКА — логика в ручке

# НЕЛЬЗЯ: импорт из соседнего сервиса если есть в /mono/repo/
from account.repo.X import Y   # ОШИБКА
from account.services.auth_service import ...  # ОШИБКА
```

---

## 2. Структура монорепы

```
/mono/
├── models/                  # SQLAlchemy-модели (единственный источник схемы БД)
│   ├── billing.py
│   ├── crm.py
│   ├── lines.py             # Line, ChannelAccount, Instance и ChannelType enum
│   ├── messages.py          # Conversation, Message, MessageMedia
│   ├── notifications.py
│   └── tenants.py           # User, Tenant, Membership
│
├── repo/                    # Общие репозитории (используются всеми сервисами)
│   ├── base.py              # BaseRepository — базовый CRUD
│   ├── billing_repo.py
│   ├── conversation_repo.py
│   ├── crm_settings_repo.py
│   ├── instance_repo.py     # InstanceRepository (НЕ в line_repo!)
│   ├── line_repo.py
│   ├── membership_repo.py
│   ├── message_repo.py
│   ├── tenant_repo.py
│   └── user.py
│
├── schemas/                 # Общие Pydantic-схемы
│   ├── auth_schemas.py
│   ├── chat_schemas.py
│   ├── line_schemas.py
│   └── tenant_schemas.py
│
├── services/                # Общие сервисы
│   ├── auth_service.py      # login, register, set_password, verify — ВСЁ про авторизацию
│   ├── billing_service.py
│   ├── cleanup_service.py
│   ├── jwt_services.py      # генерация и валидация JWT
│   ├── line_service.py
│   ├── storage_service.py   # S3/MinIO: save_file, upload_file
│   ├── tenant_services.py
│   └── user_service.py
│
├── core/                    # Общая инфраструктура
│   ├── config.py            # Settings (pydantic-settings)
│   ├── database.py          # AsyncEngine, get_db
│   ├── dependencies.py      # get_current_user, check_tenant_access
│   ├── redis.py             # Redis-клиент
│   └── security.py          # хэширование паролей, проверка
│
├── infra/                   # Внешние интеграции
│   ├── crypto.py            # encrypt/decrypt для credentials
│   ├── email/               # отправка email (SMTP)
│   └── sms/                 # отправка SMS
│
├── notification/            # Модуль уведомлений
│   ├── notification_caller.py
│   ├── notification_repo.py
│   └── notification_service.py
│
├── alembic/                 # Миграции БД
│   ├── env.py
│   └── versions/
│
├── account/                 # Сервис личного кабинета (порт 8071 / NodePort 30082)
├── admin/                   # Сервис админки (NodePort 30089 backend /adminbe/)
├── chat/                    # Чат-сервис / API (порт 8001 / NodePort 30084)
├── crm/                     # CRM-интегратор (порт 8073 / NodePort 30088)
└── airflow/                 # Планировщик задач
```

Каждый конечный сервис (`account/`, `admin/`, `chat/`, `crm/`) имеет единообразную внутреннюю структуру:

```
<service>/
├── app/
│   ├── api/
│   │   ├── main.py          # FastAPI app, подключение роутеров, startup/shutdown
│   │   └── v1/              # Роутеры по доменам
│   ├── core/
│   │   ├── dependencies.py  # DI: get_X_repo, get_X_service
│   │   └── uow.py           # UnitOfWork (если используется)
│   ├── repositories/        # Репозитории, специфичные для сервиса
│   ├── schemas/             # Схемы, специфичные для сервиса
│   └── services/            # Сервисы, специфичные для сервиса
├── Dockerfile
└── pyproject.toml
```

---

## 3. База данных и подключение

### Подключение

Общий модуль `/mono/core/database.py` создаёт асинхронный движок SQLAlchemy и фабрику сессий:

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

engine = create_async_engine(settings.DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

Зависимость `get_db` используется через `Depends` во всех ручках:

```python
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()  # явный commit при выходе
        except Exception:
            await session.rollback()
            raise
```

> В `/mono/chat/app/core/database.py` своя копия `get_db` с явным `commit` при выходе из контекста.

### Миграции

Только через Alembic. SQL руками — запрещено.

```bash
alembic revision --autogenerate -m "описание"
alembic upgrade head
```

Все модели описаны в `/mono/models/`. `alembic/env.py` импортирует `Base` и все модули моделей для автогенерации миграций.

---

## 4. Базовый репозиторий

Файл: `/mono/repo/base.py`

`BaseRepository[ModelType]` реализует универсальный CRUD для любой SQLAlchemy-модели:

```python
class BaseRepository(Generic[ModelType]):
    def __init__(self, model: type[ModelType], session: AsyncSession):
        self.model = model
        self.session = session

    async def get_by_id(self, id: Any) -> ModelType | None: ...
    async def get_multi(
        self,
        *,
        skip: int = 0,
        limit: int = 100,
        filters: dict[str, Any] | None = None,
        join_model: Any | None = None,
        order_by: list[tuple[str, str]] | None = None,
    ) -> list[ModelType]: ...
    async def create(self, **kwargs) -> ModelType: ...
    async def update(self, id: Any, **kwargs) -> ModelType | None: ...
    async def delete(self, id: Any) -> bool: ...
```

**Фильтрация в `get_multi`:**

- `filters={"field": value}` — генерирует `WHERE field = value`
- `filters={"field": [v1, v2]}` — генерирует `WHERE field IN (v1, v2)`
- `filters={"field": "%pattern%"}` — генерирует `WHERE field ILIKE '%pattern%'`
- `join_model` — позволяет фильтровать по полям связанной модели

Алхимию напрямую применять только если через `get_multi` запрос не выражается. Частые кейсы — обсуждаются и добавляются в базовый репо.

### Специфичный репозиторий

```python
# /mono/repo/membership_repo.py
class MembershipRepository(BaseRepository[Membership]):
    def __init__(self, session: AsyncSession):
        super().__init__(Membership, session)

    async def get_by_user_and_tenant(self, user_id, tenant_id): ...
    async def get_by_tenant(self, tenant_id, skip=0, limit=100, role=None): ...
    async def delete_by_user_and_tenant(self, user_id, tenant_id): ...
```

Модель `Membership` импортируется только здесь — не в сервисах, не в ручках.

---

## 5. Паттерн создания сервисов

Единый паттерн для всех сервисов: ручка → сервис → репозиторий.

### Dependencies (`dependencies.py`)

```python
def get_x_repo(db: AsyncSession = Depends(get_db)) -> XRepository:
    return XRepository(db)

def get_x_service(repo: XRepository = Depends(get_x_repo)) -> XService:
    return XService(repo)
```

### Сервис

```python
class XService:
    def __init__(self, repo: XRepository):
        self._repo = repo

    async def get_items(self) -> list[XSchema]:
        items = await self._repo.get_multi()
        return [XSchema.model_validate(i) for i in items]
```

### Ручка

```python
@router.get("/items", response_model=list[XSchema])
async def list_items(
    service: XService = Depends(get_x_service),
    current_user: UserContext = Depends(get_current_user),
):
    return await service.get_items()
```

### UnitOfWork

В сервисах, которым нужен доступ к нескольким репозиториям в одной транзакции, используется паттерн UoW:

```python
class AccountUnitOfWork:
    def __init__(self, db: AsyncSession):
        self._session = db
        self.lines = LineRepository(db)
        self.channel_accounts = ChannelAccountRepository(db)
        self.memberships = MembershipRepository(db)
        # ...
```

Сессия доступна как `self._uow._session` (не `.session`).

### Форматирование вызовов

```python
# Правильно — каждый аргумент на своей строке
result = await some_service.create(
    tenant_id=tenant_id,
    name=data.name,
    role=data.role,
)

# Неправильно — выравнивание по скобке
result = await some_service.create(tenant_id=tenant_id,
                                   name=data.name)
```

пример проброса зависимости в ручку

/mono/core/database.py

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

/mono/account/app/core/dependencies.py

def get_notification_repo(db: AsyncSession = Depends(get_db)) -> NotificationRepository:
    return NotificationRepository(db)

def get_notification_service(repo: NotificationRepository = Depends(get_notification_repo)) -> NotificationService:
    return NotificationService(repo)

/mono/account/app/api/v1/notifications.py
@router.post("/")
async def create_notification_settings(
    data: NotificationSettingsCreate,
    service: NotificationService = Depends(get_notification_service),
    current_user: UserContext = Depends(get_current_user),
):
    return await service.create(data, current_user)


---

## 6. Авторизация и роли

### Две системы ролей

В системе сосуществуют две независимые роли:

| Поле | Таблица | Значения | Назначение |
|------|---------|----------|------------|
| `user.role` | `users` | `owner`, `admin`, `support`, `user` | Системная роль, глобальный доступ |
| `membership.role` | `memberships` | `owner`, `member` | Роль в компании (тенанте) |

В JWT-токен попадают оба значения:
- `role` = `user.role`
- `member` = `membership.role`

`user.role = admin` — не ограничен ни в чём и никогда.

### JWT и контекст пользователя

Файл: `/mono/services/jwt_services.py`

```python
class TokenPayload(BaseModel):
    sub: str          # user_id
    role: str | None  # системная роль
    member: str | None  # роль в тенанте
    tenant_id: str | None

class UserContext(BaseModel):
    id: uuid.UUID
    email: str
    role: str         # UserRole
    member: str | None
```

`switch_context` при смене тенанта обновляет JWT: записывает `role=user.role`, `member=membership.role`.

### Зависимости авторизации

Файл: `/mono/core/dependencies.py`

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> UserContext:
    payload = verify_token(token)
    # валидация, получение пользователя из БД
    return UserContext(...)

async def check_tenant_access(
    tenant_id: uuid.UUID,
    current_user: UserContext = Depends(get_current_user),
) -> UserContext:
    # проверяет tenant_id и member (должен быть owner/admin/member)
    return current_user
```

### Auth-сервис

Файл: `/mono/services/auth_service.py`

Содержит всю логику авторизации: `login`, `register`, `set_password`, `verify`. Импортируется в конечные сервисы — своих копий авторизации нигде нет.

```python
# ПРАВИЛЬНО
from services.auth_service import AuthService

# ОШИБКА
from account.services.auth_service import AuthService
```

### Шифрование credentials

Файл: `/mono/infra/crypto.py`

Функции `encrypt` / `decrypt` используются для хранения токенов и ключей каналов в зашифрованном виде в поле `credentials_encrypted` (JSONB). Импортируются там где нужно расшифровать — обычно в сервисах или консьюмерах.

```python
from infra.crypto import decrypt

creds = json.loads(decrypt(channel_acc.credentials_encrypted["encrypted_data"]))
```

Правильное имя функции: `decrypt` (не `decrypt_credentials`).

---

## 7. Общие инфраструктурные модули

### Storage (S3/MinIO)

Файл: `/mono/services/storage_service.py`

Единственное место для работы с файловым хранилищем. Методы: `save_file`, `upload_file`. Импортируется везде где нужно сохранить или получить файл.

### Redis

Файл: `/mono/core/redis.py`

Используется для WebSocket pub/sub (каналы `ws:new_message:{conversation_id}`, `ws:message_status:{conversation_id}`).

### Email / SMS

Файлы: `/mono/infra/email/sender.py`, `/mono/infra/sms/sender.py`

Отправка уведомлений пользователям. Вызываются из `notification_service` или напрямую из сервисов.

### Конфигурация

Файл: `/mono/core/config.py`

`Settings` на базе `pydantic-settings`, читает переменные окружения. Единый объект `settings` импортируется везде:

```python
from core.config import settings
```

### Логирование

После каждого внешнего вызова (httpx, Kafka, внешние API) — обязательный `log.info`:

```python
resp = await client.post(url, json=payload)
log.info(f"external_call method=POST url={url} status={resp.status_code}")
```

---

## Стек технологий

| Слой | Технология |
|------|-----------|
| Язык | Python 3.12 |
| Web-фреймворк | FastAPI |
| ORM | SQLAlchemy 2.0 async |
| БД | PostgreSQL 16 (драйвер asyncpg) |
| Кэш / pub-sub | Redis |
| Очереди | Apache Kafka |
| Хранилище файлов | MinIO (S3-совместимый) |
| Контейнеризация | Docker + k3s (Kubernetes) |
| Reverse proxy | Nginx |
| Планировщик | Airflow 3.0 |
| Шаблоны (CRM) | Jinja2 |
| Миграции | Alembic |





==========================================================
**********************************************************
==========================================================



# account-service

> Сервис личного кабинета DoveChat. Порт **8071**, NodePort **30082**.  
> Архитектурные паттерны (UoW, BaseRepository, get_db, роли) — см. `dovechat-architecture.md`.

---

## Назначение

Управляет всем, что касается конфигурации тенанта:

- Регистрация, авторизация, смена пароля
- Управление тенантами и участниками команды
- Линии (очереди обращений) и подключение каналов связи
- Биллинг и тарифы
- Настройки уведомлений и вебхуков

---

## Структура

```
account/
├── app/
│   ├── api/v1/
│   │   ├── auth.py
│   │   ├── billing.py
│   │   ├── channels.py
│   │   ├── lines.py
│   │   ├── members.py
│   │   ├── notifications.py
│   │   ├── tenant.py
│   ├── core/
│   │   ├── dependencies.py   # get_X_repo / get_X_service
│   │   └── uow.py            # AccountUnitOfWork
│   ├── repositories/
│   │   ├── billing_repo.py
│   │   ├── instance_repo.py
│   │   ├── line_repo.py
│   │   ├── membership_repo.py
│   │   ├── notification_repo.py
│   │   ├── permission_repo.py
│   │   └── tenant_repo.py
│   ├── schemas/
│   │   ├── billing_schemas.py
│   │   ├── channel_schemas.py
│   │   ├── crm_settings_schemas.py
│   │   ├── line_schemas.py
│   │   ├── member_schemas.py
│   │   ├── notification_schemas.py
│   │   └── tenant_schemas.py
│   └── services/
│       ├── billing_service.py
│       ├── channel_service.py
│       ├── crm_settings_service.py
│       ├── line_service.py
│       ├── member_service.py
│       ├── notification_service.py
│       ├── tenant_services.py
│       └── webhook_service.py
```

---

## Импорты

Авторизация и общая логика берётся из корня монорепы — account-service не дублирует:

| Что | Откуда |
|-----|--------|
| `login`, `register`, `set_password`, `verify` | `/mono/services/auth_service.py` |
| Работа с S3/MinIO | `/mono/services/storage_service.py` |
| `MembershipRepository`, `TenantRepository`, `UserRepository`, `InstanceRepository`, `BillingRepository` | `/mono/repo/` |
| Модели | `/mono/models/` |
| Общие схемы | `/mono/schemas/` |

Собственные репозитории, схемы и сервисы — только account-специфичные.

---

## Паттерн ручек

```
get_db()  →  get_X_repo(db=Depends(get_db))  →  get_X_service(repo=Depends(get_X_repo))
                                                         ↓
                                              router endpoint(service=Depends(get_X_service), user=Depends(get_current_user))
```

- Модели — только в репо
- Логика — только в сервисе
- Ручка — только вызов сервиса

---

## Авторизация

Из `/mono/core/dependencies.py`:

- `get_current_user` → валидирует JWT, возвращает `UserContext(id, email, role, member)`
- `check_tenant_access` → проверяет `tenant_id` + `member`

### Две системы ролей

| Поле | Таблица | Значения | Смысл |
|------|---------|----------|-------|
| `user.role` | `users` | `owner`, `admin`, `support`, `user` | Системная роль |
| `membership.role` | `memberships` | `owner`, `member` | Роль в компании |

В JWT: `role = user.role`, `member = membership.role`.  
`user.role = admin` — не ограничен ничем.

---

## Модули API

### auth.py — Аутентификация

Регистрация, логин, смена пароля. Делегирует в `/mono/services/auth_service.py`.

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/auth/register` | Регистрация нового пользователя |
| `POST` | `/auth/login` | Логин, возвращает JWT |
| `POST` | `/auth/set-password` | Смена пароля |
| `POST` | `/auth/verify` | Верификация (email/токен) |

---

### tenant.py — Тенант

Управление данными компании.

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/tenant/{id}` | Получить тенанта |
| `PATCH` | `/tenant/{id}` | Обновить (name, address, phone, email, inn) |
| `GET` | `/tenants` | Список тенантов пользователя |

`PATCH` использует `model_dump(exclude_unset=True)` — обновляет только переданные поля.  
Тенант расширен полями: `address`, `phone`, `email`, `inn`.

---

### members.py — Участники команды

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/tenant/{id}/members` | Список участников |
| `POST` | `/tenant/{id}/members` | Пригласить участника |
| `PATCH` | `/tenant/{id}/members/{user_id}` | Изменить роль |
| `DELETE` | `/tenant/{id}/members/{user_id}` | Удалить участника |

`membership.role` в JWT не обновляется автоматически при смене роли — пользователю нужно перелогиниться.

Модель `Membership`:
```python
class Membership(Base):
    __tablename__ = "memberships"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID, ForeignKey("users.id", ondelete="CASCADE"))
    tenant_id = Column(UUID, ForeignKey("tenants.id", ondelete="CASCADE"))
    role = Column(String, default="staff")
    is_active = Column(Boolean, default=True)
    # relationship на User — отсутствует
```

---

### lines.py — Линии

Линия = очередь обращений, к которой привязан канал.

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/lines` | Список линий тенанта |
| `POST` | `/lines` | Создать линию |
| `GET` | `/lines/{id}` | Получить линию |
| `PATCH` | `/lines/{id}` | Обновить линию |
| `DELETE` | `/lines/{id}` | Удалить линию |

---

### channels.py — Каналы связи

Подключение и управление аккаунтами каналов.

#### Типы каналов (`ChannelType`, `/mono/models/lines.py`)

| Тип | Описание |
|-----|----------|
| `telegram_bot` | Telegram Bot API |
| `telegram_user` | Telegram MTProto через tg-connector |
| `vk` | ВКонтакте |
| `waba` | WhatsApp Business API (360dialog) |
| `whatsapp_green` | WhatsApp через Green API |

Креды хранятся зашифрованными в `channel_accounts.credentials_encrypted` (JSONB).

#### Расшифровка кредов

```python
from infra.crypto import decrypt
creds = json.loads(decrypt(channel_acc.credentials_encrypted["encrypted_data"]))
```

#### WhatsApp Green — флоу подключения

1. `POST /account/whatsapp-green/connect` — берёт свободный инстанс, получает QR от Green API
2. `POST /account/whatsapp-green` — сохраняет креды, устанавливает вебхук если переданы `id_instance` и `api_token`

`_set_green_webhook` вызывается **только** при явной передаче кредов.

Вебхук направляется на chat-сервис:
```
https://tg.dove-chat.com/dev/api/v1/webhooks/whatsapp/green/{line_id}
```

---

### billing.py — Биллинг

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/billing/plans` | Список тарифов |
| `GET` | `/billing/balance` | Текущий баланс тенанта |
| `POST` | `/billing/pay` | Пополнение счёта |
| `GET` | `/billing/history` | История: платежи + корректировки ledger в одном списке `BillingHistoryItem` |

---

### notifications.py — Уведомления

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/notifications/settings` | Получить настройки |
| `PATCH` | `/notifications/settings` | Обновить настройки |
| `GET` | `/notifications/history` | История уведомлений |

---

### webhooks.py — Вебхуки

POST /webhooks/whatsapp/green/{line_id} Вебхук от Green API (в т.ч. после сканирования QR)
POST /webhooks/vk/{line_id} Вебхук от ВКонтакте
Обработка делегируется в WebhookService.process_green_webhook / process_vk_webhook.

VK-эндпоинт возвращает PlainTextResponse (требование VK для подтверждения).

---

## UnitOfWork

`AccountUnitOfWork` в `account/app/core/uow.py`.  
Сессия доступна как `self._uow._session` (не `.session`).  
commit/rollback/close — только внутри `get_db`, никогда в UoW или репо.

---

## Зависимости (dependencies.py)

```python
# Паттерн для каждого репо/сервиса:
def get_line_repo(db: AsyncSession = Depends(get_db)) -> LineRepository:
    return LineRepository(db)

def get_line_service(repo: LineRepository = Depends(get_line_repo)) -> LineService:
    return LineService(repo)
```

---

## Стек

Python 3.12, FastAPI, SQLAlchemy 2.0 async, asyncpg, PostgreSQL 16, Redis, MinIO, Docker, k3s

---

## Тесты

```
tests/
├── integration/
│   ├── test_auth.py
│   ├── test_billing_api.py
│   ├── test_channel_api.py
│   ├── test_line_api.py
│   ├── test_member_api.py
│   ├── test_notification_api.py
│   ├── test_tenant_api.py
│   ├── test_waba.py
│   └── test_whatsapp_green_api.py
└── unit/
    ├── test_auth_service.py
    ├── test_billing_service.py
    ├── test_channel_service.py
    ├── test_line_service.py
    ├── test_members.py
    ├── test_notification_service.py
    ├── test_tenant_service.py
    └── test_whatsapp_green.py
```


==========================================================
**********************************************************
==========================================================


# chat-service

> Сервис обмена сообщениями DoveChat.  
> Архитектурные паттерны — см. `dovechat-architecture.md`.

---

## Назначение

- Приём входящих сообщений от всех каналов (через Kafka consumers и вебхуки)
- Отправка исходящих сообщений (через Kafka → senders)
- Хранение разговоров и сообщений
- Проксирование в CRM (Bitrix24) через crm-service
- Realtime-доставка через WebSocket
- Обработка медиафайлов (MinIO/S3)

---

## Структура

```
chat/
├── app/
│   ├── api/v1/
│   │   ├── chat.py         # REST эндпоинты
│   │   ├── webhooks.py     # вебхуки от каналов
│   │   └── websocket.py    # WS соединения
│   ├── consumers/          # Kafka consumers (по каналу)
│   │   ├── telegram_consumer.py
│   │   ├── telegram_listener.py
│   │   ├── telegram_listener_service.py
│   │   ├── outgoing_consumer.py
│   │   ├── green_whatsapp.py
│   │   ├── vk.py
│   │   ├── waba.py
│   │   └── dlt_consumer.py
│   ├── core/
│   │   ├── dependencies.py
│   │   ├── kafka_producer.py
│   │   ├── redis_subscriber.py
│   │   └── uow.py
│   ├── repositories/
│   │   ├── chat_repo.py
│   │   ├── line_repo.py
│   │   ├── message_repo.py
│   │   └── permission_repo.py
│   ├── schemas/
│   │   ├── chat_schemas.py
│   │   ├── message_schemas.py
│   │   └── telegram_schemas.py
│   ├── senders/            # Отправка по каналу
│   │   ├── base.py
│   │   ├── factory.py
│   │   ├── telegram_bot.py
│   │   ├── telegram_user.py
│   │   ├── vk.py
│   │   ├── waba.py
│   │   └── whatsapp_green.py
│   ├── services/
│   │   ├── bot_service.py
│   │   ├── chat_service.py
│   │   ├── line_service.py
│   │   ├── webhook_service.py
│   │   ├── websocket_manager.py
│   │   └── websocket_service.py
│   └── utils.py
```

---

## API (chat.py)

### Исходящие сообщения

Все исходящие идут через `_enqueue_outgoing`:
1. `chat_service.conv_msg(data, user)` — сохраняет сообщение в БД
2. `kafka_producer.send_message(...)` — отправляет в Kafka
3. Возвращает `MessageResponse`

| Метод | Путь | Канал |
|-------|------|-------|
| `POST` | `/messages/tgbot` | `telegram_bot` |
| `POST` | `/messages/tguser` | `telegram_user` |
| `POST` | `/messages/whatsapp/green` | `whatsapp_green` |
| `POST` | `/messages/waba` | `waba` |
| `POST` | `/messages/vk` | `vk` |

Все принимают `multipart/form-data`: `data: str` (JSON MessageCreate) + `files: list[UploadFile]`.

### Внутренние эндпоинты (без JWT)

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/internal/crm-outgoing` | Исходящее из Bitrix24 (contact card). `UserContext` собирается из form-полей `user_id`/`user_email` |
| `POST` | `/internal/crm-message` | Входящее из CRM → `chat_service.save_crm_incoming(CrmMessageIn)` |
| `POST` | `/internal/crm-robot-send` | Bizproc robot → `chat_service.robot_send(CrmMessageOut)` |

### Разговоры

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/conversations` | Список с фильтрами: `line_id`, `channel_type`, `is_closed` |
| `GET` | `/conversations/{id}` | Детали + сообщения |
| `POST` | `/conversations` | Создать разговор |
| `PATCH` | `/conversations/{id}/status` | Открыть/закрыть (`is_closed: bool`) |
| `GET` | `/conversations/{id}/messages` | Сообщения (пагинация: `skip`, `limit`) |

### Линии

| Метод | Путь | Описание |
|-------|------|----------|
| `GET` | `/lines` | Линии пользователя (с учётом роли) |
| `GET` | `/lines/{id}/channels` | Каналы линии |
| `GET` | `/lines-with-channels` | Линии со списком доступных каналов |

### Прочее

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/upload` | Загрузка файла в S3. Принимает `media_id: UUID` + `file: UploadFile` |
| `POST` | `/bot/webhook/{line_id}` | Вебхук Telegram Bot API → `BotService.handle_webhook` |

---

## WebSocket (websocket.py)

Авторизация: JWT передаётся query-параметром `token`. Декодируется локально через `settings.secret_key`.

| Путь | Описание |
|------|----------|
| `ws://.../chat/{conversation_id}?token=` | Подписка на конкретный разговор |
| `ws://.../tenant/{tenant_id}?token=` | Подписка на все события тенанта |
| `ws://.../user?token=` | Подписка на события пользователя |

При `tenant_id` в JWT не совпадает с `tenant_id` в пути — закрывает с кодом `1008`.

`manager.connect` получает: `websocket`, `conversation_id`, `user_id`, `tenant_id`, `role`.  
`manager.connect_tenant` используется для `/tenant/` и `/user/` (по `user_id`).

---

## BotService (bot_service.py)

Обрабатывает входящие апдейты от Telegram Bot API.

### handle_webhook(line_id, update_data)

1. Получает `Line` по `line_id`
2. Извлекает `chat_id`, `text`, наличие медиа
3. `conv_repo.get_or_create_by_chat_id(...)` — находит или создаёт разговор
4. Расшифровывает креды канала: `decrypt_credentials(channel_acc.credentials_encrypted)` → `bot_token`
5. При наличии медиа — `_process_media_message(...)` → скачивает с TG, сохраняет в S3
6. `msg_repo.create_message(direction="inbound", ...)`
7. При новом разговоре — `manager.send_new_conversation(user_id, ...)` всем участникам тенанта
8. `manager.send_new_message(conversation_id, ws_message)`
9. `crm_service.send_to_crm(...)` → получает `bx_chat_id`
10. Если `bx_chat_id` получен — `conv_repo.update(conversation, crm_chat_id=bx_chat_id)`

### _process_media_message(message, bot_token, raw_content)

Поддерживает: `photo` (берёт последний — наибольший), `document`.  
Флоу: `getFile` → скачать → `storage.save_file(bytes, media_id, file_name, mime_type)`.  
Возвращает: `(media_id, file_name, mime_type, raw_content, file_size)`.

---

## WebhookService (webhook_service.py)

Обрабатывает вебхуки от Green API (WhatsApp) и VK.

### process_green_webhook(payload)

1. Определяет `event_type` из `typeWebhook`
2. Извлекает `idInstance` → `instance_repo.get_line_by_instance_id(instance_id)` → `line`
3. Извлекает `chatId` → нормализует в `phone` (`+...`)
4. `chat_repo.get_conversations_by_phone(line_id, phone, "whatsapp_green")` → `conv`
5. `WebhookHandlerFactory.get_handler(event_type)` → делегирует обработку

### Обработчики событий

| Тип события | Обработчик | Действие |
|-------------|-----------|----------|
| `incomingMessageReceived` | `IncomingMessageHandler` | `chat_service.save_incoming_green(conv_id, payload)` |
| `outgoingMessageStatus` | `MessageStatusHandler` | `chat_service.update_message_status(external_id, status)` |
| `stateInstanceChanged` / `statusInstanceChanged` | `StateChangedHandler` | Обновляет `connection_status`, `phone` в channel_accounts при `authorized` |

---

## Senders

`SenderFactory.get_sender(channel_type)` → sender. Все наследуют `BaseSender(ABC)`: абстрактный `send(data: MessageCreate) -> dict`, опциональный `extract_msg_id(result) -> str | None`.

| channel_type | Класс | Инициализация | Медиа |
|---|---|---|---|
| `telegram_bot` | `TelegramBotSender` | `set_bot_token(token)` | `sendPhoto` / `sendDocument` по mime; файл из S3 через `get_file_from_s3` |
| `telegram_user` | `TelegramUserSender` | `set_listener(listener)` | делегирует в `listener.send_file_user(data)` |
| `whatsapp_green` | `WhatsAppGreenSender` | `set_creds({"id_instance", "api_token"})` | URL: `s3_external_url + s3_bt_bucket + media_id` |
| `waba` | `WabaSender` | `set_creds({"api_key"})` | URL: `s3_external_url + media_id` |
| `vk` | `VKSender` | `set_creds({"access_token"})` | скачивает из `s3_internal_url`, загружает через `upload_and_send_photo` |

`data.phone` везде передаётся без `+` (`.lstrip('+')`), кроме `telegram_bot` — там `chat_id = data.phone` как есть.

---

## Kafka

### Producer (kafka_producer.py)

TODO: уточнить сигнатуру по коду `core/kafka_producer.py`.

### Consumers

#### outgoing_consumer.py
Читает исходящие из Kafka → `SenderFactory`. При ошибке (> `MAX_RETRIES`) → пишет в `DLT_TOPIC`.  
Методы: `process_message`, `start_consumer`.

#### dlt_consumer.py
Обрабатывает Dead Letter Topic (`DLT_TOPIC`). Повторные попытки через `RETRY_INTERVAL`, до `MAX_RETRIES`. Проверка наличия следующих сообщений через `_has_more`.  
Методы: `process_dlt_message`, `start_dlt_consumer`, `_has_more`.

#### telegram_consumer.py
Входящие от tg-connector (MTProto). Управляет клиентами через `_clients`.  
Методы: `get_or_create_client`, `start_consumer`.

#### telegram_listener.py
`TelegramConnection` — соединение MTProto для одного чата: `connect`, `run`, `disconnect`, `handler`, `_handle_message`, `wait_for_client`. Переподключение: `_reconnect_delay`, `_connection_error_notified`.

`TelegramListener` — управляет пулом соединений (`clients`, `bots`, `tasks`):
- `start_listening` / `stop_listening`
- `send_message_user(data)` / `send_file_user(data)` — исходящие через MTProto
- `send_message_bot(data)` — исходящие через Bot API
- `_save_and_notify` — сохранение + WS
- `_flush_album` / `_album_buffer` / `_album_tasks` — сборка медиагрупп (album)

**Известный баг:** `send_file_user` — double-path в MinIO URL.

#### telegram_listener_service.py
`TelegramListenerService` — оркестрирует `TelegramListener`: запускает/останавливает прослушку по чатам, управляет Kafka control consumer.  
Методы: `start_listening_for_chat`, `stop_listening_for_chat`, `_listen_chat_task`, `handler`, `_handle_incoming_message`, `_kafka_control_consumer`, `start`, `shutdown`.

#### green_whatsapp.py
`GreenWhatsAppConnection` — HTTP-клиент Green API: `send_message`, `send_file`, `send` (общий).  
`GreenWhatsAppListener` — обрабатывает входящие вебхуки: `handle_webhook`, `_handle_incoming_message`, `_handle_message_status`, `_handle_instance_state`, `send_message`.

#### vk.py
`VkConnection` — HTTP-клиент VK API (`VK_API_URL`, `VK_API_VERSION`): `call`, `send_message`, `send_file`, `upload_and_send_photo`.  
`VkListener` — входящие + отправка: `handle_webhook`, `vk_handle_reply`, `vk_confirm`, `_get_creds`, `_handle_incoming_message`, `_download_file`, `send_message`, `send_file`.

#### waba.py
`WabaConnection` — HTTP-клиент 360dialog: `send_message`, `send_file`.  
`WabaListener` — входящие: `handle_webhook`, `_handle_incoming_message`, `_handle_message_status`, `download_media`, `send_message`, `send_file`.

---

## Схемы (chat_schemas.py)

### MessageCreate
```python
id: UUID  # генерируется по умолчанию
conversation_id: UUID
direction: MessageDirection  # inbound | outbound
phone: str | None
text: str | None
media: list[dict] | None  # [{"file_name": ..., "mime_type": ...}]
channel_msg_id: str | None
status: MessageStatus = "sent"
```

### MessageResponse
```python
id: int
conversation_id: UUID
direction: MessageDirection
status: MessageStatus
text: str | None
created_at: datetime
user_id: UUID | None
user_email: str | None
user_full_name: str | None
media: list[MediaResponse]
```

### ConversationListResponse
```python
id: UUID
line_id: UUID
line_name: str | None
tenant_id: UUID | None
channel_type: str
crm_chat_id: str
client_phone: str | None
unread_count: int
last_message: MessageResponse | None
```

### CrmMessageIn
```python
line_id: UUID
crm_chat_id: str
channel_type: str
text: str
raw_content: dict
media: list[dict]
user_id: UUID
user_email: str
```

### CrmMessageOut
```python
line_id: UUID
client_phone: str
text: str
crm_connection_id: UUID
contact_id: str
crm_line_id: str
connector_id: str
channel_type: str
channel_account_id: UUID | None
media: list[dict]
```

---

## Схемы Kafka (telegram_schemas.py)

| Схема | Назначение |
|-------|-----------|
| `KafkaEvent` | Базовый: `event_id`, `timestamp`, `account_phone`, `payload` |
| `IncomingMessageEvent` | Входящее: `chat_id`, `message_id`, `sender_id`, `text`, `media_info`, `raw_update` |
| `MessageStatusEvent` | Статус: `chat_id`, `message_id`, `status`, `error_details` |
| `ContactSyncEvent` | Контакт: `chat_id`, `chat_type`, `title`, `username`, `phone_number` |
| `TelegramOutgoingMessage` | Исходящее: `user_id`, `session_string`, `recipient`, `message` |

---

## Интеграция с CRM

Флоу входящего сообщения → CRM:
```
handle_webhook / consumer
    → chat_service.save / msg_repo.create_message
    → crm_service.send_to_crm(tenant_id, conversation_id, ...) → bx_chat_id
    → conv_repo.update(conversation, crm_chat_id=bx_chat_id)
```

Фильтрация системных сообщений BT: `user_id == "0"` → игнорировать в `handle_bitrix_incoming`.

Флоу исходящего из CRM:
```
POST /internal/crm-outgoing  →  _enqueue_outgoing  →  Kafka  →  SenderFactory
```

Флоу Bizproc robot:
```
POST /internal/crm-robot-send  →  chat_service.robot_send(CrmMessageOut)
```

---

## Стек

Python 3.12, FastAPI, SQLAlchemy 2.0 async, Kafka (aiokafka), Redis, MinIO, WebSocket, k3s




==============================================================
**************************************************************
==============================================================
# Документация: CRM-сервис (`/mono/crm`)

> Актуально на 17.06.2026. Описывает только CRM-интегратор. Инфраструктура (nginx, k8s) — в деплой-доке.

---

## Содержание

1. [Обзор сервиса](#1-обзор-сервиса)
2. [Структура файлов](#2-структура-файлов)
3. [Конфигурация и настройки](#3-конфигурация-и-настройки)
4. [Слой данных](#4-слой-данных)
   - 4.1 [Модели](#41-модели)
   - 4.2 [Репозитории](#42-репозитории)
   - 4.3 [Unit of Work](#43-unit-of-work-crmuow)
5. [Схемы (Pydantic)](#5-схемы-pydantic)
6. [API — эндпоинты](#6-api--эндпоинты)
   - 6.1 [install_router — установка и авторизация](#61-install_router--установка-и-авторизация)
   - 6.2 [webhooks_router — входящие вебхуки](#62-webhooks_router--входящие-вебхуки)
   - 6.3 [outgoing_router — исходящие](#63-outgoing_router--исходящие)
   - 6.4 [connections_router — управление подключениями](#64-connections_router--управление-подключениями)
   - 6.5 [robots_router — роботы Битрикс24](#65-robots_router--роботы-битрикс24)
7. [Бизнес-логика: CrmService](#7-бизнес-логика-crmservice)
   - 7.1 [Установка и авторизация](#71-установка-и-авторизация)
   - 7.2 [Входящие сообщения из Битрикс24](#72-входящие-сообщения-из-битрикс24)
   - 7.3 [Исходящие сообщения](#73-исходящие-сообщения)
   - 7.4 [Управление подключениями](#74-управление-подключениями)
   - 7.5 [Роботы Битрикс24](#75-роботы-битрикс24)
8. [Провайдер Битрикс24](#8-провайдер-битрикс24)
   - 8.1 [BitrixClient](#81-bitrixclient)
   - 8.2 [BitrixProvider](#82-bitrixprovider)
9. [Флоу установки — сквозное описание](#9-флоу-установки--сквозное-описание)
10. [Флоу входящего сообщения — сквозное описание](#10-флоу-входящего-сообщения--сквозное-описание)
11. [Пробелы в документации](#11-пробелы-в-документации)

---

## 1. Обзор сервиса

CRM-сервис — отдельный FastAPI-микросервис в монорепе (`/mono/crm`). Отвечает за:

- установку приложения DoveChat в Битрикс24 (OAuth-подобный флоу через iframe);
- регистрацию open-channel коннекторов в Битрикс24;
- приём входящих сообщений из Битрикс24 и трансляцию их в chat-сервис;
- отправку исходящих сообщений из Битрикс24 клиенту;
- управление роботами бизнес-процессов Битрикс24 (`dovechat_send_text`, `dovechat_send_file`, `dovechat_check_account`);
- CRUD подключений CRM в ЛК.

Сервис слушает на порту **30088**, проксируется nginx по префиксу `/crm/`.

Зависимости на другие сервисы монорепы:

| Сервис | Адрес | Для чего |
|--------|-------|---------|
| chat-сервис | `http://chat.dchat.svc.cluster.local:8001` | трансляция входящих/исходящих |
| tg-connector | `http://tg-connector.dchat.svc.cluster.local:8000` | проверка телефона в Telegram |
| Битрикс24 REST API | `https://{domain}/rest/` | регистрация коннекторов, роботов, события |
| Green API (WhatsApp) | `https://api.green-api.com/` | проверка телефона в WhatsApp |

---

## 2. Структура файлов

```
crm/
├── app/
│   ├── api/
│   │   ├── main.py                  # FastAPI app, подключение роутеров
│   │   └── v1/
│   │       ├── install.py           # install_router + static_router
│   │       ├── webhooks.py          # webhooks_router
│   │       ├── outgoing.py          # outgoing_router
│   │       ├── connections.py       # connections_router
│   │       └── robots.py            # robots_router
│   ├── core/
│   │   ├── config.py                # settings (BaseSettings)
│   │   ├── database.py              # get_db, AsyncSession
│   │   ├── dependencies.py          # get_uow, get_crm_service (FastAPI DI)
│   │   └── uow.py                   # CrmUnitOfWork
│   ├── repositories/
│   │   └── crm_repo.py              # CrmConnectionRepository,
│   │                                #   CrmDialogLinkRepository,
│   │                                #   CrmSettingsRepository
│   ├── schemas/
│   │   └── crm_schemas.py           # CrmConnectRequest, CrmConnectionResponse, …
│   ├── services/
│   │   ├── crm_service.py           # CrmService — вся бизнес-логика
│   │   └── providers/
│   │       ├── base.py              # BaseCrmProvider (абстракция)
│   │       └── bitrix/
│   │           ├── client.py        # BitrixClient (HTTP к Bitrix REST)
│   │           └── provider.py      # BitrixProvider
└── html/                            # Jinja2 / статические HTML-файлы
    ├── login.html
    ├── lines.html
    ├── success.html
    ├── install_finish.html
    └── contact_tab.html
```

---

## 3. Конфигурация и настройки

Файл: `crm/app/core/config.py` (содержимое не получено, известны используемые поля).

| Поле | Тип | Назначение |
|------|-----|-----------|
| `host_url` | `str` | Публичный URL сервиса — вставляется в login.html и в redirect-URL |
| `BITRIX_CONNECTOR_MAP` | `dict[str, str]` | Маппинг `channel_type → connector_id` |
| `BITRIX_CONNECTOR_NAMES` | `dict[str, str]` | Маппинг `connector_id → отображаемое имя` |

### BITRIX_CONNECTOR_MAP

```python
{
    "telegram_bot":      "dovechat_telegram_bot",
    "telegram_user":     "dovechat_telegram_user",
    "vk":                "dovechat_vk",
    "whatsapp_green":    "dovechat_whatsapp_green",
    "waba":              "dovechat_waba",
}
```

### BITRIX_CONNECTOR_NAMES

```python
{
    "dovechat_telegram_bot":    "DoveChat TgBot",
    "dovechat_telegram_user":   "DoveChat TgUser",
    "dovechat_vk":              "DoveChat VK",
    "dovechat_whatsapp_green":  "DoveChat WA",
    "dovechat_waba":            "DoveChat WABA",
}
```

---

## 4. Слой данных

### 4.1 Модели

Модели живут в общей части монорепы (`/mono/models/`). CRM-сервис использует:

| Модель | Таблица | Откуда |
|--------|---------|--------|
| `CrmConnection` | `crm_connections` | `/mono/models/crm.py` (предположительно) |
| `CrmDialogLink` | `crm_dialog_links` | то же |
| `CrmSettings` | `crm_settings` | то же |
| `ChannelAccount` | `channel_accounts` | `/mono/models/lines.py` |
| `Line` | `lines` | `/mono/models/lines.py` |
| `Conversation` | `conversations` | `/mono/models/chat.py` (предположительно) |
| `Membership` | `memberships` | `/mono/models/tenants.py` |
| `User` | `users` | `/mono/models/users.py` (предположительно) |

#### CrmConnection — известные поля

| Поле | Тип | Описание |
|------|-----|---------|
| `id` | UUID | PK |
| `member_id` | str | Идентификатор установки в Битрикс24 |
| `connector_id` | str | ID коннектора (из `BITRIX_CONNECTOR_MAP`) |
| `line_id` | UUID | FK → `lines.id` |
| `tenant_id` | UUID | FK → `tenants.id` |
| `channel_type` | str | Тип канала (`telegram_bot`, `whatsapp_green`, …) |
| `is_active` | bool | Флаг активности |
| `access_token` | str | OAuth access token Битрикс24 |
| `refresh_token` | str | OAuth refresh token Битрикс24 |
| `domain` | str | Домен портала Битрикс24 |

> ⚠️ Полная схема модели не получена — поля выведены из использования в `crm_service.py`.

#### Membership (из `/mono/models/tenants.py`)

```python
class Membership(Base):
    __tablename__ = "memberships"
    id         = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id    = Column(UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    tenant_id  = Column(UUID(as_uuid=True), ForeignKey("tenants.id", ondelete="CASCADE"), nullable=False)
    role       = Column(String, nullable=False, default="staff")
    is_active  = Column(Boolean, default=True, nullable=False)
    # relationship на User отсутствует
```

### 4.2 Репозитории

Файл: `crm/app/repositories/crm_repo.py`

#### CrmConnectionRepository

Наследует `BaseRepository[CrmConnection]`.

```python
async def get_by_member_id(
    self,
    member_id: str,
) -> CrmConnection | None
```
Фильтр: `member_id=member_id, is_active=True`. Возвращает первую запись или `None`.

---

```python
async def get_by_line_and_connector(
    self,
    line_id: uuid.UUID,
    connector_id: str,
) -> CrmConnection | None
```
Фильтр: `line_id=line_id, connector_id=connector_id`. Без фильтра по `is_active`.

---

```python
async def get_by_member_and_connector(
    self,
    member_id: str,
    connector_id: str,
) -> CrmConnection | None
```
Фильтр: `member_id=member_id, connector_id=connector_id, is_active=True`. Используется при входящих вебхуках и в `connect_line`.

#### CrmDialogLinkRepository

> ⚠️ Полное содержимое не получено. Известно только имя класса.

#### CrmSettingsRepository

> ⚠️ Полное содержимое не получено. Известно только имя класса.

### 4.3 Unit of Work (`CrmUoW`)

Файл: `crm/app/core/uow.py`

```python
class CrmUnitOfWork:
    def __init__(self, db: AsyncSession):
        self.crm_connections  = CrmConnectionRepository(db)
        self.crm_dialog_links = CrmDialogLinkRepository(db)
        self.crm_settings     = CrmSettingsRepository(db)
        self.channel_accounts = BaseRepository(ChannelAccount, db)
        self.lines            = BaseRepository(Line, db)
        self.conversations    = BaseRepository(Conversation, db)
        self.memberships      = BaseRepository(Membership, db)
        self.users            = BaseRepository(User, db)
```

UoW не реализует транзакционный контекст-менеджер — сессия управляется через FastAPI DI (`get_db`).

---

## 5. Схемы (Pydantic)

### Файл: `crm/app/schemas/crm_schemas.py`

> ⚠️ Файл не получен. Известны только схемы, используемые в роутерах.

#### CrmConnectRequest

Используется в `POST /connections/{tenant_id}`. Точные поля неизвестны.

#### CrmConnectionResponse

Используется в `POST /connections/{tenant_id}` и `GET /connections/{tenant_id}`. Точные поля неизвестны.

### Файл: `chat/app/schemas/chat_schemas.py` (используется в CRM-сервисе)

#### MessageCreate

```python
class MessageCreate(BaseModel):
    id:              uuid.UUID      = Field(default_factory=uuid.uuid4)
    message_id:      int | None     = None
    conversation_id: uuid.UUID
    channel_msg_id:  str | None     = None
    direction:       MessageDirection
    phone:           str | None     = None
    text:            str | None     = None
    media:           list[dict] | None = None
    raw_content:     dict | None    = None
    status:          MessageStatus  = "sent"
```

#### CrmMessageIn

```python
class CrmMessageIn(BaseModel):
    line_id:      uuid.UUID
    crm_chat_id:  str
    channel_type: str
    text:         str
    raw_content:  dict       = {}
    media:        list[dict] = []
```

---

## 6. API — эндпоинты

### 6.1 `install_router` — установка и авторизация

Файл: `crm/app/api/v1/install.py`  
Префикс роутера: `/install`  
Итоговые пути (с учётом глобального префикса `/crm/api/v1`): `/crm/api/v1/install/...`

Также в этом файле объявлен `static_router` с префиксом `/html` для раздачи статических HTML-файлов.

---

#### `GET /install/bitrix`

```python
@install_router.get("/bitrix")
async def bitrix_placement_get(request: Request)
```

Рендерит `login.html` через Jinja2 с передачей переменной `host_url = settings.host_url`.  
Вызывается Битрикс24 при открытии iframe настроек коннектора.

---

#### `POST /install/bitrix`

```python
@install_router.post("/bitrix")
async def bitrix_placement_post(
    request: Request,
    crm_service: CrmService = Depends(get_crm_service),
)
```

Читает тело запроса как `bytes`, извлекает `DOMAIN` из query params.  
Делегирует в → `crm_service.handle_onappinstall(body, domain)`.  
Возвращает `RedirectResponse` (303).

---

#### `POST /install/bitrix/onappinstall`

```python
@install_router.post("/bitrix/onappinstall")
async def bitrix_onappinstall(
    request: Request,
    crm_service: CrmService = Depends(get_crm_service),
)
```

Дублирует логику `POST /install/bitrix`. Используется как альтернативный endpoint при регистрации события `OnAppInstall` в Битрикс24.  
Делегирует в → `crm_service.handle_onappinstall(body, domain)`.

---

#### `POST /install/bitrix/auth`

```python
@install_router.post("/bitrix/auth")
async def bitrix_auth(
    body: dict,
    crm_service: CrmService = Depends(get_crm_service),
)
```

**Тело запроса:**
| Поле | Тип | Обязательно |
|------|-----|------------|
| `email` | str | да |
| `password` | str | да |
| `bx_auth` | dict | нет (default `{}`) |

Делегирует в → `crm_service.get_lines_for_bitrix(email, password, bx_auth)`.

---

#### `POST /install/bitrix/select-tenant`

```python
@install_router.post("/bitrix/select-tenant")
async def bitrix_select_tenant(
    body: dict,
    crm_service: CrmService = Depends(get_crm_service),
)
```

**Тело запроса:**
| Поле | Тип | Обязательно |
|------|-----|------------|
| `jwt_token` | str | да |
| `tenant_id` | str | да |
| `bx_auth` | dict | нет (default `{}`) |

Делегирует в → `crm_service.get_lines_for_tenant(jwt_token, tenant_id, bx_auth)`.

---

#### `POST /install/bitrix/connect`

```python
@install_router.post("/bitrix/connect")
async def bitrix_connect(
    body: dict,
    crm_service: CrmService = Depends(get_crm_service),
)
```

**Тело запроса:**
| Поле | Тип | Обязательно |
|------|-----|------------|
| `line_id` | str (UUID) | да |
| `bx_auth` | dict | нет (default `{}`) |
| `channel_account_id` | str (UUID) | нет |

Делегирует в → `crm_service.connect_line(line_id, bx_auth)`.

---

#### Статические HTML-маршруты (`static_router`, префикс `/html`)

| Метод | Путь | Файл |
|-------|------|------|
| GET/POST | `/html/lines.html` | `lines.html` |
| GET/POST | `/html/success.html` | `success.html` |
| GET/POST | `/html/login.html` | `login.html` |
| GET/POST | `/html/install_finish.html` | `install_finish.html` |
| GET/POST | `/html/html/contact-tab.html` | `contact_tab.html` |

> ⚠️ Путь `/html/html/contact-tab.html` выглядит как дублирование префикса — возможно, баг в роутере.

---

### 6.2 `webhooks_router` — входящие вебхуки

Файл: `crm/app/api/v1/webhooks.py`  
Префикс: `/webhooks`

---

#### `POST /webhooks/{crm_type}`

```python
@webhooks_router.post("/{crm_type}")
async def crm_webhook(
    crm_type: str,
    request: Request,
    uow: CrmUnitOfWork = Depends(get_uow),
)
```

Принимает вебхук от Битрикс24. `crm_type` в пути не используется в логике (принимается любое значение). Тело парсится как `application/x-www-form-urlencoded`.

Поток:
1. Читает `body = await request.body()`
2. Парсит через `parse_qs` → `dict[str, str]`
3. Логирует RAW body и распарсенный dict
4. `CrmService(uow).handle_bitrix_incoming(data)`
5. Возвращает `{"status": "ok"}`

---

#### `GET|POST /webhooks/install/bitrix`

```python
@webhooks_router.api_route("/install/bitrix", methods=["GET", "POST"])
async def bitrix_install()
```

Возвращает минимальный HTML с вызовом `BX24.installFinish()`. Служит как fallback-страница завершения установки (альтернатива `install_finish.html`).

---

### 6.3 `outgoing_router` — исходящие

Файл: `crm/app/api/v1/outgoing.py`  
Префикс: `/outgoing`

> ⚠️ DI в этом файле реализован вручную (не через `get_crm_service` из `dependencies.py`): `CrmService(uow)` создаётся прямо в handler'е. Расхождение с другими роутерами.

---

#### `GET /outgoing/lines`

```python
@outgoing_router.get("/lines")
async def get_lines_for_member(
    member_id: str,
    uow: CrmUnitOfWork = Depends(get_uow),
)
```

**Query params:** `member_id: str`  
Делегирует в → `crm_service.get_lines_for_member(member_id)`.

---

#### `POST /outgoing/start`

```python
@outgoing_router.post("/start")
async def start_outgoing(
    body: dict,
    uow: CrmUnitOfWork = Depends(get_uow),
)
```

**Тело запроса:**
| Поле | Тип | Обязательно |
|------|-----|------------|
| `member_id` | str | да |
| `contact_id` | str | да |
| `line_id` | str (UUID) | да |
| `channel_account_id` | str (UUID) | нет |
| `text` | str | нет (default `""`) |

Делегирует в → `crm_service.start_outgoing(member_id, contact_id, line_id, channel_account_id, text)`.

---

#### `GET|POST /outgoing/contact-tab`

```python
@outgoing_router.api_route("/contact-tab", methods=["GET", "POST"])
async def contact_tab()
```

Возвращает `FileResponse` для `contact_tab.html`.

---

### 6.4 `connections_router` — управление подключениями

Файл: `crm/app/api/v1/connections.py`  
Префикс: `/connections`

---

#### `POST /connections/{tenant_id}`

```python
@connections_router.post("/{tenant_id}", response_model=CrmConnectionResponse)
async def connect_crm(
    tenant_id: uuid.UUID,
    body: CrmConnectRequest,
    crm_service: CrmService = Depends(get_crm_service),
)
```

Создаёт или обновляет CRM-подключение для тенанта.  
Делегирует в → `crm_service.connect(tenant_id, body)`.

---

#### `DELETE /connections/{tenant_id}/{crm_type}`

```python
@connections_router.delete("/{tenant_id}/{crm_type}")
async def disconnect_crm(
    tenant_id: uuid.UUID,
    crm_type: str,
    crm_service: CrmService = Depends(get_crm_service),
)
```

Отключает CRM для тенанта по типу.  
Делегирует в → `crm_service.disconnect(tenant_id, crm_type)`.  
Возвращает `{"status": "ok"}`.

---

#### `GET /connections/{tenant_id}`

```python
@connections_router.get("/{tenant_id}", response_model=list[CrmConnectionResponse])
async def get_connections(
    tenant_id: uuid.UUID,
    crm_service: CrmService = Depends(get_crm_service),
)
```

Возвращает список активных подключений для тенанта.  
Делегирует в → `crm_service.get_connections(tenant_id)`.

---

### 6.5 `robots_router` — роботы Битрикс24

Файл: `crm/app/api/v1/robots.py`  
Префикс: `/robots`

> ⚠️ Тело файла не получено. Известна только сигнатура и поведение из `crm_service.py`.

#### `POST /robots/{robot_code}`

Принимает вызов от Битрикс24 при срабатывании робота в бизнес-процессе.  
Делегирует в → `crm_service.handle_robot(robot_code, data)`.  
Всегда возвращает `{"status": "ok"}`.

**Зарегистрированные роботы:**
| `robot_code` | Назначение |
|--------------|-----------|
| `dovechat_send_text` | Отправить текстовое сообщение клиенту |
| `dovechat_send_file` | Отправить файл клиенту |
| `dovechat_check_account` | Проверить наличие телефона в мессенджере |

> `bizproc.robot.add` доступен только на платных тарифах Битрикс24. На бесплатном тарифе возвращает 404 — это штатное поведение платформы, не ошибка кода.

---

## 7. Бизнес-логика: CrmService

Файл: `crm/app/services/crm_service.py`

```python
class CrmService:
    def __init__(self, uow: CrmUnitOfWork):
        self._uow = uow
```

### 7.1 Установка и авторизация

#### `handle_onappinstall`

```python
async def handle_onappinstall(
    self,
    body: bytes,
    domain: str = "",
) -> RedirectResponse
```

Вызывается при POST-запросе от Битрикс24 в момент установки приложения.

**Шаги:**
1. `await self.save_install_tokens(body, domain=domain)` — сохраняет токены.
2. Парсит `body` через `parse_qs`.
3. Если `PLACEMENT == "SETTING_CONNECTOR"`:
   - Извлекает `domain`, `access_token`, `refresh_token`, `member_id` из тела.
   - Извлекает `connector` из JSON-поля `PLACEMENT_OPTIONS["CONNECTOR"]`.
   - Возвращает `RedirectResponse(303)` на `GET /crm/api/v1/install/bitrix` с параметрами в query string.
4. Иначе: возвращает `RedirectResponse(303)` на `/crm/html/install_finish.html`.

---

#### `save_install_tokens`

```python
async def save_install_tokens(
    self,
    body: bytes,
    domain: str = "",
) -> None
```

**Шаги:**
1. Ищет существующую запись `conn` в `crm_connections` по `member_id`.
2. Если `conn` существует — обновляет только токены (`access_token`, `refresh_token`).
3. Если `conn` не существует — **не создаёт** запись (заготовка удалена). Запись создаётся только в `connect_line`.
4. Регистрирует все коннекторы из `settings.BITRIX_CONNECTOR_MAP` через `BitrixClient.register_connector`. Ошибки регистрации логируются как `warning`, не прерывают флоу.

---

#### `get_lines_for_bitrix`

```python
async def get_lines_for_bitrix(
    self,
    email: str,
    password: str,
    bx_auth: dict,
) -> dict
```

Авторизует пользователя DoveChat по `email`/`password`. Возвращает список тенантов для выбора (если их несколько) или сразу список линий.

> ⚠️ Полная реализация не получена.

---

#### `get_lines_for_tenant`

```python
async def get_lines_for_tenant(
    self,
    jwt_token: str,
    tenant_id: str,
    bx_auth: dict,
) -> dict
```

Возвращает список линий для выбранного тенанта. `jwt_token` используется для проверки сессии.

> ⚠️ Полная реализация не получена.

---

#### `connect_line`

```python
async def connect_line(
    self,
    line_id: uuid.UUID,
    bx_auth: dict,
) -> dict
```

Привязывает линию DoveChat к коннектору Битрикс24.

**Шаги:**
1. `connector_id = bx_auth.get("connector")`.
2. Определяет `channel_type_val` обратным маппингом по `BITRIX_CONNECTOR_MAP`. Если коннектор не найден — возвращает `{"error": "Неподдерживаемый коннектор: {connector_id}"}`.
3. Ищет существующую запись через `get_by_member_and_connector(member_id, connector_id)`.
4. Если запись есть — обновляет. Если нет — **создаёт новую** `CrmConnection`.

> Создание записи `crm_connections` происходит **только здесь**. В `save_install_tokens` заготовка не создаётся.

---

### 7.2 Входящие сообщения из Битрикс24

#### `handle_bitrix_incoming`

```python
async def handle_bitrix_incoming(
    self,
    data: dict,
) -> None
```

Обрабатывает входящее сообщение из открытого канала Битрикс24.

**Шаги:**
1. Извлекает `member_id = data.get("data[USER_ID]")` (или аналог) и `connector_id = data.get("data[CONNECTOR]")`.
2. `conn = await self._uow.crm_connections.get_by_member_and_connector(member_id, connector_id)`.
3. Получает `owner` тенанта:
   ```python
   memberships = await self._uow.memberships.get_multi(
       filters={"tenant_id": conn.tenant_id, "role": "owner"},
       limit=1,
   )
   owner = await self._uow.users.get_by_id(memberships[0].user_id)
   ```
4. Получает или создаёт `Conversation`.
5. Собирает вложения: `files = await self._collect_bitrix_files(data)`.
6. Формирует `MessageCreate` с `direction=outbound`, `status=sent`.
7. POST на `http://chat.dchat.svc.cluster.local:8001/v1/internal/crm-outgoing`:
   - Если есть файлы — `multipart/form-data` с полями `data`, `files[]`, `channel_type`, `user_id`, `user_email`.
   - Если файлов нет — `application/x-www-form-urlencoded` с полями `data`, `channel_type`, `user_id`, `user_email`.

---

#### `_collect_bitrix_files`

```python
async def _collect_bitrix_files(
    self,
    data: dict,
) -> list[tuple[bytes, str, str]]
```

Скачивает файлы из вебхука Битрикс24. Итерирует по индексам `i = 0, 1, 2, …` пока существует ключ `data[MESSAGES][0][message][files][{i}][downloadLink]`.

Для каждого файла: скачивает через `httpx.AsyncClient.get(link)`, добавляет `(content: bytes, name: str, mime: str)` в результат.  
Ошибки скачивания логируются как `error`, файл пропускается.

---

### 7.3 Исходящие сообщения

#### `get_lines_for_member`

```python
async def get_lines_for_member(
    self,
    member_id: str,
) -> dict
```

Возвращает линии, доступные для исходящего обращения по `member_id` (из `crm_connections`).

> ⚠️ Полная реализация не получена.

---

#### `start_outgoing`

```python
async def start_outgoing(
    self,
    member_id: str,
    contact_id: str,
    line_id: uuid.UUID,
    channel_account_id: uuid.UUID | None,
    text: str,
) -> dict
```

Инициирует исходящий диалог из Битрикс24 к клиенту.

> ⚠️ Полная реализация не получена.

---

### 7.4 Управление подключениями

#### `connect`

```python
async def connect(
    self,
    tenant_id: uuid.UUID,
    body: CrmConnectRequest,
) -> CrmConnectionResponse
```

> ⚠️ Реализация не получена.

---

#### `disconnect`

```python
async def disconnect(
    self,
    tenant_id: uuid.UUID,
    crm_type: str,
) -> None
```

> ⚠️ Реализация не получена.

---

#### `get_connections`

```python
async def get_connections(
    self,
    tenant_id: uuid.UUID,
) -> list[CrmConnectionResponse]
```

> ⚠️ Реализация не получена.

---

### 7.5 Роботы Битрикс24

#### `handle_robot`

```python
async def handle_robot(
    self,
    robot_code: str,
    data: dict,
) -> dict
```

Диспетчер. Маршрутизирует по `robot_code` на конкретный метод. Всегда возвращает `{"status": "ok"}` — Битрикс24 не ожидает результата синхронно.

---

#### `_robot_send_text`

> ⚠️ Сигнатура не получена. Отправляет текстовое сообщение клиенту через chat-сервис.

---

#### `_robot_send_file`

> ⚠️ Сигнатура не получена. Отправляет файл клиенту через chat-сервис.

---

#### `_robot_check_account`

```python
async def _robot_check_account(
    self,
    phone: str,
    line_id: uuid.UUID,
) -> dict  # {"exists": "Y" | "N"}
```

Проверяет наличие телефонного номера в мессенджере. Логика зависит от типа канала:

**Ветка Telegram (`telegram_user`):**
1. Ищет `ChannelAccount` с `line_id=line_id, channel_type=telegram_user`.
2. POST `http://tg-connector.dchat.svc.cluster.local:8000/prod/check-phone` с `{"phone", "session_string"}`.
3. `exists = "Y" if result["exists"] else "N"`.

**Ветка WhatsApp (`whatsapp_green`):**
1. Если `telegram_user` не найден — ищет `ChannelAccount` с `channel_type=whatsapp_green`.
2. Если аккаунт не найден — `{"exists": "N", "user_id": "", "error": "Нет активного канала"}`.
3. Декриптует `credentials_encrypted["encrypted_data"]` через `infra.crypto.decrypt`.
4. POST `https://api.green-api.com/waInstance{id_instance}/checkWhatsapp/{api_token}` с `{"phoneNumber": int(phone)}`.
5. `exists = "Y" if result["existsWhatsapp"] else "N"`.

Возвращает только `{"exists": "Y" | "N"}` (поля `user_id` и `error` во внешний ответ не пробрасываются).

---

## 8. Провайдер Битрикс24

### 8.1 BitrixClient

Файл: `crm/app/services/providers/bitrix/client.py`

> ⚠️ Получен только метод `call`. Конструктор и другие методы не получены.

```python
class BitrixClient:
    domain: str
    access_token: str
```

#### `call`

```python
async def call(
    self,
    method: str,
    params: dict,
    use_json: bool = False,
) -> dict
```

Выполняет запрос к Bitrix24 REST API.

- URL: `https://{self.domain}/rest/{method}.json`
- Добавляет `params["auth"] = self.access_token`
- Если `use_json=True` — POST как JSON (`Content-Type: application/json`)
- Если `use_json=False` — POST как form-data
- Таймаут: 30 сек
- При HTTP-ошибке: логирует `error`, бросает исключение через `raise_for_status()`

---

#### Известные вызовы `BitrixClient` из кода

| Метод Битрикс24 | Вызов в коде | Назначение |
|----------------|-------------|-----------|
| `imconnector.register` | `bx.register_connector(connector_id, name)` | Регистрация open-channel коннектора |
| `bizproc.robot.add` | (из `save_install_tokens`) | Регистрация роботов в бизнес-процессах |
| `event.bind` | (из `save_install_tokens`) | Подписка на события Битрикс24 |

> ⚠️ Полный список методов недоступен — `provider.py` и полный `client.py` не получены.

---

### 8.2 BitrixProvider

Файл: `crm/app/services/providers/bitrix/provider.py`

> ⚠️ Файл не получен.

Предположительно наследует `BaseCrmProvider` из `providers/base.py` и использует `BitrixClient` для взаимодействия с REST API Битрикс24.

---

## 9. Флоу установки — сквозное описание

```
Битрикс24 (iframe)
    │
    ├─ GET /crm/api/v1/install/bitrix
    │      └─ Jinja2: рендер login.html
    │         (hostUrl = settings.host_url передаётся в JS)
    │
    │  [пользователь вводит email/password]
    │
    ├─ POST /crm/api/v1/install/bitrix/auth
    │      └─ CrmService.get_lines_for_bitrix(email, password, bx_auth)
    │         └─ возвращает JWT + список тенантов или сразу линии
    │
    │  [если несколько тенантов — пользователь выбирает]
    │
    ├─ POST /crm/api/v1/install/bitrix/select-tenant
    │      └─ CrmService.get_lines_for_tenant(jwt_token, tenant_id, bx_auth)
    │         └─ возвращает список линий
    │
    │  [пользователь выбирает линию]
    │
    ├─ POST /crm/api/v1/install/bitrix/connect
    │      └─ CrmService.connect_line(line_id, bx_auth)
    │         ├─ определяет connector_id и channel_type из bx_auth["connector"]
    │         ├─ get_by_member_and_connector → update или create CrmConnection
    │         └─ возвращает результат
    │
    │  [BX24 вызывает OnAppInstall]
    │
    └─ POST /crm/api/v1/install/bitrix  (или /bitrix/onappinstall)
           └─ CrmService.handle_onappinstall(body, domain)
              ├─ save_install_tokens:
              │    ├─ обновляет токены если conn существует
              │    └─ регистрирует все коннекторы + роботов + события
              └─ если PLACEMENT=SETTING_CONNECTOR:
                   └─ RedirectResponse → GET /install/bitrix?...&CONNECTOR={id}
                 иначе:
                   └─ RedirectResponse → /crm/html/install_finish.html
```

**Хранение состояния между шагами в браузере:**
- `login.html` сохраняет `bx_auth` (включая `connector`) и `host_url` в `sessionStorage`
- `lines.html` читает `host_url` из `sessionStorage`

---

## 10. Флоу входящего сообщения — сквозное описание

```
Битрикс24
    │
    └─ POST /crm/api/v1/webhooks/{crm_type}
           │  (body: application/x-www-form-urlencoded)
           │
           └─ CrmService.handle_bitrix_incoming(data)
              │
              ├─ crm_connections.get_by_member_and_connector(member_id, connector_id)
              │    └─ conn: CrmConnection
              │
              ├─ memberships.get_multi(tenant_id=conn.tenant_id, role="owner")
              │    └─ users.get_by_id(membership.user_id)
              │         └─ owner: User
              │
              ├─ получить или создать Conversation
              │
              ├─ _collect_bitrix_files(data)
              │    └─ httpx.get(downloadLink) для каждого файла
              │
              ├─ MessageCreate(direction=outbound, status=sent, …)
              │
              └─ POST http://chat.dchat.svc.cluster.local:8001/v1/internal/crm-outgoing
                   ├─ если files: multipart/form-data
                   └─ если нет files: form-data (data + channel_type + user_id + user_email)
```

---

## 11. Пробелы в документации

Следующие файлы не были получены и требуют дополнения:

| Файл | Что неизвестно |
|------|---------------|
| `crm/app/core/dependencies.py` | Реализация `get_crm_service`, `get_uow` |
| `crm/app/core/config.py` | Полный список полей `Settings` |
| `crm/app/schemas/crm_schemas.py` | Поля `CrmConnectRequest`, `CrmConnectionResponse` и других схем |
| `crm/app/repositories/crm_repo.py` | `CrmDialogLinkRepository`, `CrmSettingsRepository` (полные классы) |
| `crm/app/services/crm_service.py` | `get_lines_for_bitrix`, `get_lines_for_tenant`, `get_lines_for_member`, `start_outgoing`, `connect`, `disconnect`, `get_connections`, `handle_robot` (диспетчер), `_robot_send_text`, `_robot_send_file` |
| `crm/app/services/providers/base.py` | `BaseCrmProvider` |
| `crm/app/services/providers/bitrix/client.py` | Конструктор, все методы кроме `call` |
| `crm/app/services/providers/bitrix/provider.py` | Весь файл |
| `crm/app/api/v1/robots.py` | Весь файл (тело пустое) |
| Модели `CrmConnection`, `CrmDialogLink`, `CrmSettings` | Полные поля таблиц |
