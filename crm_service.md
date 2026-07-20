# Оглавление
## Техническая документация
## Инструкция по установке 


# DoveChat CRM-модуль — техническая документация

Путь: `/opt/dovechat/mono/crm/`
Дата актуальности: 18 июля 2026 (после схемного рефактора CrmConnection → CrmAccount + CrmLineConnection)

---

## 1. Структура

Структура CRM-модуля аналогична проочим сервисом, основной код находится в папке /mono/crm, не считая общих фрагментов - базовый репозиторий, конфигурация и тд.
Общие модели/схемы — в `/mono/models/`, `/mono/schemas/`.
Свои - `app/repositories/`, `app/services/`, `app/api/`. 

Подключение роутеров: `/opt/dovechat/mono/crm/app/api/main.py`
- `install_router` → `prefix="/api/v1"`, сам роутер → `prefix="/install"`
- `robots_router` → маршруты `/robots/{robot_code}`

nginx: `location /crm/ { proxy_pass http://127.0.0.1:30088/; proxy_set_header Host $host; }` — единственный проверенный location для CRM-сервиса.

---

## 2. Модели данных

### CrmAccount (`crm_accounts`) — уровень аккаунта/портала
Одна запись = один Bitrix-портал или один amoCRM-аккаунт, привязанный к тенанту.
Поля: `id, tenant_id, crm_type ("bitrix24"|"amocrm"), domain, access_token_enc, refresh_token_enc, is_active, token_updated_at, created_at, member_id (Bitrix), amojo_id (amo), account_id (amo), no_deal_action, auto_create_contact, responsible_user_id`.

### CrmLineConnection (`crm_line_connections`) — уровень линии/канала
Одна запись = один DoveChat line, подключённый к одному Bitrix-коннектору на аккаунте выше.
Поля: `id, crm_account_id (FK crm_accounts), line_id (FK lines), channel_type, crm_line_id, connector_id, is_active, created_at`.


### CrmUnitOfWork (`/mono/crm/app/core/uow.py`)
```python
self.crm_accounts = CrmAccountRepository(db)   # физически работает с CrmAccount!
self.crm_dialog_links = CrmDialogLinkRepository(db)
self.crm_settings = CrmSettingsRepository(db)
self.crm_line_connections = CrmLineConnectionRepository(db)
self.channel_accounts = BaseRepository(ChannelAccount, db)
self.lines = BaseRepository(Line, db)
self.conversations = BaseRepository(Conversation, db)
self.memberships = BaseRepository(Membership, db)
self.users = BaseRepository(User, db)
```

### crm_chat_id — актуальная схема
conversations.crm_chat_id — числовой Bitrix CHAT_ID сессии. Берётся из ответа imconnector.send.messages → result.DATA.RESULT[0].session.CHAT_ID, пишется в send_to_crm при каждой отправке. По нему handle_bitrix_incoming ищет диалог на входящих.
crm_dialog_links для Bitrix не используется. Используется только для amoCRM.

---

## 3. Установка приложения Bitrix24

### Install-роуты (`/opt/dovechat/mono/crm/app/api/v1/install.py`)
- `GET /install/bitrix` — отдаёт `login.html`
- `POST /install/bitrix` — `crm_service.handle_onappinstall(body, domain)`
- `POST /install/bitrix/onappinstall` — дублирующий роут на тот же handler
- `POST /install/bitrix/auth` — `crm_service.get_lines_for_bitrix(email, password, bx_auth)`
- `POST /install/bitrix/connect` — `crm_service.connect_line(line_id, bx_auth)`
- `POST /install/bitrix/select-tenant` — `crm_service.get_lines_for_tenant(jwt_token, tenant_id, bx_auth)`

### Install URL
```
https://dev.dove-chat.pro/crm/api/v1/install/bitrix
```
Складывается: nginx `/crm/` → FastAPI `prefix="/api/v1"` → роутер `prefix="/install"` → `/bitrix`.

### Секреты приложения
`BITRIX_APPLICATION_TOKEN`, `BITRIX_CLIENT_ID`, `BITRIX_CLIENT_SECRET` — глобальные env-константы, единые для всех клиентов. При смене портала/новом клиенте — обновлять вручную. Мультитенантное хранение (таблица типа `BitrixAppCredentials`) не реализовано — открытая задача.

### Флоу onappinstall
Bitrix вызывает `onappinstall` дважды подряд в рамках одной установки. На первом вызове возможна `ERROR_ACTIVITY_NOT_FOUND` при регистрации роботов — самоустраняется на втором вызове.

### install_finish.html
Белый экран после install — не баг: файл содержит только JS (`BX24.callMethod("crm.userfield.add", ...)` → `BX24.installFinish()`), видимого UI не предполагается. `installFinish()` вызывается независимо от результата `userfield.add`.

### Подтверждение успешной установки
`/devops/list/` — приложение в списке "Интеграции" со scope: CRM, Чат и уведомления, Открытые линии, Коннекторы для внешних мессенджеров, Продажи в чате, Пользователи, Встраивание приложений, Настройки пользовательских полей, Бизнес-процессы.
В Открытых линиях — 5 коннекторов: `DoveChat TgBot`, `DoveChat TgUser`, `DoveChat WA`, `DoveChat WABA`, `DoveChat VK`.

---

## 4. Привязка линии (connect_line)

`/mono/crm/app/services/crm_service.py`, метод `connect_line(line_id, bx_auth)`:

1. Ищет существующий `CrmAccount` по `domain` через `crm_accounts.get_multi(filters={"domain": domain})`.
2. Рефрешит токены через `BitrixClient.refresh_tokens`.
3. Создаёт/обновляет `CrmAccount` (`crm_accounts.create`/`.update`) с зашифрованными токенами, `member_id`.
4. Резолвит `channel_type` из `bx_auth["connector"]` через `settings.BITRIX_CONNECTOR_MAP`.
6. Ищет существующий `CrmLineConnection` по `line_id`+`connector_id`.
7. Получает `crm_line_id` (Bitrix open line id) через `bx.create_open_line(line.name)`, если ещё нет.
9. Создаёт/обновляет `CrmLineConnection` со связкой `crm_account_id`.

---

## 5. Роботы Bitrix (bizproc)

### Регистрация — `_register_robots(self, bx)` в `crm_service.py`
Вызывается при onappinstall (после успешного install). Для каждого из трёх кодов:
- `dovechat_send_text`
- `dovechat_send_file`
- `dovechat_check_account`

Шаги на каждый робот:
1. `bizproc.robot.delete` (CODE) — на случай если уже существует, ошибки игнорируются.
2. `bizproc.robot.add` с `HANDLER = "{settings.host_url}/crm/api/v1/robots/{robot_code}"`, `AUTH_USER_ID: 1`, `USE_SUBSCRIPTION: "Y"`, `PROPERTIES` (phone, line_id, text/file_url/file_name в зависимости от типа), `RETURN_PROPERTIES` (status/message_id/error или exists/user_id/error для check_account), `FILTER.INCLUDE`: `CCrmDocumentDeal`, `CCrmDocumentLead`, `CCrmDocumentContact`.

После роботов — отдельно `crm.userfield.add` для поля `DOVECHAT_WA_TG_STATUS` на `CRM_DEAL` (в try/except, `ERROR_METHOD_NOT_FOUND` на некоторых тарифах не блокирует install).

### Проверка регистрации
Роботы НЕ видны отдельным списком в интерфейсе Bitrix. Проверка:
- REST: `bizproc.robot.list` (через access_token или входящий вебхук) — возвращает массив CODE зарегистрированных роботов.
- В интерфейсе: карточка сделки → Роботы и триггеры → +Создать → в модалке список роботов → прокрутить до группы "Другие" → в самом низу списка справа — три dovechat_* робота. Их нужно вручную добавить в бизнес-процесс сделки, чтобы они реально срабатывали.

### Обработка вызова робота — `robots_router`
```
POST /robots/{robot_code}
```
Bitrix шлёт `application/x-www-form-urlencoded`, парсится через `parse_qs`. Хендлер вызывает `crm_service.handle_robot(robot_code, data)`.

### RobotJob (`robot_jobs`, `/mono/models/`)
Таблица очереди/статуса запуска джобов роботов (не факта регистрации в Bitrix).
Поля: `id, tenant_id, job_type (send_text|send_file|check_account), status (RobotJobStatus enum), params (JSON), result (JSON), started_at, completed_at, created_at`.

---

## 6. Поток сообщений Bitrix

### Исходящее (DoveChat → Bitrix): `send_to_crm` в `crm_service.py`
1. Получает `conn` (CrmAccount, доджойненный с line-полями через `get_by_tenant_and_channel`) и вызывает `provider.send_message(...)`.
2. Ветка amoCRM: `crm_chat_id = result.get("crm_chat_id", f"dc_{conversation_id}")`, пишет в `conversations.crm_chat_id`.
3. Ветка Bitrix24: достаёт `session_chat_id` из `result["result"]["DATA"]["RESULT"][0]["session"]["CHAT_ID"]`, если отличается от текущего — обновляет `conversations.crm_chat_id`.
4. Дополнительно (Bitrix): ищет контакт по телефону (`bx.find_contact_by_phone`), если найден — ищет открытую сделку (`bx.find_open_deal`), если есть — привязывает чат к сделке (`bx.attach_deal_to_chat`).
5. Легаси-путь через `crm_dialog_links`/`dc_{conversation_id}` для Bitrix полностью убран (актуально с сегодняшнего рефактора).

### Исходящее — сборка сообщения: `BitrixClient.send_message`
- `text` не может быть пустым при наличии файлов — если пусто, подставляется `" "` (иначе Bitrix отклоняет весь запрос `IMCONNECTOR_NOT_ALL_THE_REQUIRED_DATA`).
- Медиа передаются как `message.files = [{"url": ..., "name": ...}]`, `url` обязан быть presigned-ссылкой (через `StorageService.presigned_url(media_id)`), не сырым путём на приватный MinIO-бакет.

### Входящее (Bitrix → DoveChat): `handle_bitrix_incoming`
1. `conn = crm_line_connections.get_by_member_and_connector(member_id, connector_id)` (репозиторий `CrmConnectionRepository`, возвращает `CrmLineConnection`, доджойненный через `CrmAccount`).
2. Догружает `account = crm_accounts.get_by_id(conn.crm_account_id)`.
3. Ищет диалог: `conversations.get_multi(filters={"line_id": conn.line_id, "crm_chat_id": crm_chat_id})`.
4. Обновление токенов при рефреше — через `crm_accounts.update(account, ...)`.
5. Событие Bitrix: `ONIMCONNECTORMESSAGEADD`.

---

## 7. Webhook signature (Bitrix)

`verify_webhook` — HMAC-SHA1, обязателен `body.strip()` перед подсчётом подписи. На новом портале (`b24-iv0fmm.bitrix24.ru`) секрет для подписи ещё не перепроверялся.


## 9. amoCRM — статус

### Widget (iframe embed в карточке сделки)
- `EmbedChat.jsx` рендерит `ConversationPage` без `Shell`/`Sidebar`.
- JWT-авторизация через `$authorizedAjax` с `AMO_CLIENT_SECRET` (HS256).
- `get_conv_by_lead` в `CrmService` — маппинг `lead_id` → `conversation_id` через кеш, затем через amoCRM API поиск по телефону.
- `amo_lead_id` добавлен в `CrmDialogLink` через Alembic-миграцию.
- Widget JS (`script.js`): `setInterval`-based вставка DOM в `.card-widgets`; кнопка-тоггл "DChat Widget" (`#1375ABFF`); iframe: `position: fixed, right: 265px, width: 66vw, height: 100vh, zIndex: 9999`.
- Эндпоинты `crm-message`/`crm-outgoing` — неиспользуемые остатки раннего подхода, не убраны.
- `handleConnectAmo` — всегда берёт `amoAccounts[0].id` — потенциальный баг при нескольких аккаунтах, не исправлено.
- Модерация виджета в маркетплейсе amoCRM отклонена; приватная установка не заблокирована.
- Последний известный блокер: `access_token` amoCRM истекал, `get_conv_by_lead` упирался в partner rate limit (400).

### Роуты amo в install.py
```
GET  /install/amo               # редирект на OAuth Kommo/amoCRM
GET  /install/amo/callback      # OAuth callback → handle_amo_install
GET  /install/amo/auth-url      # отдаёт JSON с готовым OAuth URL
POST /install/amo/save-tokens   # ручной сейв токенов, шифрование через infra.crypto.encrypt
```
`amo_save_tokens`: `GET https://{domain}/api/v4/account?with=amojo_id` с `Authorization: Bearer {access_token}`, сохраняет `domain`, `account_id`, `amojo_id`, зашифрованные токены, `tenant_id` через `crm_connections.upsert_amo`.

### Архитектура amo
- `scope_id = {channel_id}_{amojo_id}`.
- `AMO_CHANNEL_ID`, `AMO_CHANNEL_SECRET`, `amojo_id` — в конфиге.
- OAuth `/oauth2/authorize` возвращает 405 для amojo-channel интеграций — постоянный флоу это ручная форма токенов + `/install/amo/save-tokens`.
- `verify_webhook` требует `body.strip()` перед HMAC-SHA1 (аналогично Bitrix).

### Инфраструктура доставки виджета (fe_main)
- `vite dev` заброшен, только `vite build` + `npx serve dist`.
- `VITE_BASE_PATH=/dev/` убран.
- nginx: `location /assets/` и `location /embed/` → `localhost:9999`.
- `location /dev/` → `/crm/` цепочка используется для AJAX в `script.js`.

---

## 10. Инфраструктура (общее)

- Namespace `dchat`, hostPath-монтирование кода во все поды — force-delete пода подхватывает изменения без пересборки образа.
- Локальный Docker registry `localhost:5000`, теги `:1`.
- Секреты — `dchat-env-secret`.
- nginx — отдельный Docker-контейнер `nginx-1`, вне k3s.
  - `location /crm/ { proxy_pass http://127.0.0.1:30088/; proxy_set_header Host $host; }`
- MinIO: сервис `minio`, presigned URLs с TTL для media (`StorageService.presigned_url`). Единой точки формирования этих ссылок в проекте нет — каждый модуль вызывает `StorageService` по-своему.



# =======================================================
              УСТАНОВКА
# =======================================================



# Установка DoveChat в Bitrix24 — инструкция с нуля

## 1. Регистрация портала Bitrix24

1. Зарегистрируйте новый портал Bitrix24 (b24-xxxxxx.bitrix24.ru) — подходит как бесплатный, так и платный тариф.
2. Войдите в портал под администратором.

## 2. Создание локального приложения

1. Перейдите в **Разработчикам → Другое → Локальное приложение** (или `/devops/edit/application/`).
2. Укажите:
   - **Название**: DoveChat (или любое понятное название)
   - **Путь установки (install URL)**:
     ```
     https://dev.dove-chat.pro/crm/api/v1/install/bitrix
     ```
3. В разделе **Права доступа** отметьте полный набор:
   - CRM
   - Чат и уведомления (im)
   - Открытые линии (imopenlines)
   - Коннекторы для внешних мессенджеров (imconnector)
   - Продажи в чате (salescenter)
   - Пользователи (user)
   - Встраивание приложений (placement)
   - Настройки пользовательских полей (userfieldconfig)
   - Бизнес-процессы (bizproc)

   Неполный набор прав приведёт к ошибкам `401 insufficient_scope` при регистрации коннекторов и `400 ERROR_EVENT_NOT_FOUND` при подписке на события.

4. Сохраните приложение и запустите установку.

## 3. Что происходит при установке автоматически

После сохранения Bitrix дважды вызывает `onappinstall` на указанный install URL:

- На первом вызове система создаёт (или обновляет) запись аккаунта по домену портала.
- Регистрируются 5 коннекторов: Telegram Bot, Telegram User, WhatsApp (Green API), WABA, VK.
- Подключается событие `ONIMCONNECTORMESSAGEADD` (обработка входящих сообщений из открытых линий).
- Регистрируются 3 робота бизнес-процессов: `dovechat_send_text`, `dovechat_send_file`, `dovechat_check_account`.
- На первом вызове возможна ошибка `ERROR_ACTIVITY_NOT_FOUND` при регистрации роботов — это ожидаемо, она самоустраняется на втором автоматическом вызове `onappinstall`.

Проверить, что права выставлены полностью, можно в **Разработчикам → Список приложений** — напротив приложения должен быть указан полный набор прав.

## 4. Ввод Client ID и Client Secret

Bitrix не передаёт `client_id` и `client_secret` автоматически при установке — их нужно скопировать вручную:

1. В настройках приложения (страница редактирования локального приложения на портале) найдите поля **Client ID** и **Client Secret**.
2. Скопируйте оба значения.
3. В личном кабинете DoveChat откройте карточку компании → **Настройки CRM** → **Настройки Bitrix**.
4. Вставьте скопированные значения в поля **Bitrix Client ID** и **Bitrix Client Secret**, нажмите **Сохранить**.

Эти значения нужны для автоматического обновления токена доступа портала — без них при истечении токена интеграция перестанет работать.

## 5. Привязка линии DoveChat к порталу

1. В Bitrix24 перейдите в **CRM → Клиенты → Контакт-центр**.
2. Найдите коннектор DoveChat с нужным каналом (Telegram, WhatsApp и т.д.), нажмите **Создать линию**.
3. В открывшейся форме нажмите кнопку продолжения настройки вверху.
4. Откроется окно авторизации DoveChat — войдите под своей учётной записью.
5. Выберите компанию, затем нужную линию DoveChat.
6. После успешной привязки появится надпись **«Линия подключена»** с зелёной галкой.

Повторите "5. Привязка линии" для каждого канала (Telegram, WhatsApp и т.д.), который нужно подключить к CRM.

## 6. Проверка работы

1. **Исходящее сообщение**: отправьте сообщение клиенту из DoveChat — оно должно появиться в открытой линии Bitrix24.
2. **Входящее сообщение**: ответьте клиенту из интерфейса Bitrix24 — сообщение должно дойти клиенту через исходный канал (Telegram/WhatsApp/VK).
3. **Медиафайлы**: отправьте фото с обеих сторон — должно доходить в обе стороны.
4. **Роботы бизнес-процессов**: в карточке сделки откройте раздел **«Роботы и триггеры» → «+Создать»**, в списке роботов прокрутите группу **«Другие»** до конца списка — там находятся роботы `dovechat_send_text`, `dovechat_send_file`, `dovechat_check_account`. Добавьте нужный робот в бизнес-процесс сделки вручную — только тогда он будет фактически срабатывать.

## 7. Если что-то не работает

- Если приложение установилось, но коннекторы/события не регистрируются — проверьте набор прав в шаге 2 и переустановите приложение.
- Если после какого-то времени интеграция перестаёт получать/отправлять сообщения — проверьте, что Client ID/Client Secret введены в личном кабинете DoveChat (шаг 4), это необходимо для автообновления токена.
- В остальных случаях обратитесь в поддержку DoveChat.
