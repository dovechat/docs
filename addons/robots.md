# DoveChat × Bitrix24 — bizproc-роботы

## 1. Как добавить и пользоваться (для менеджера/заказчика)

### Доступные роботы
- **DoveChat: Отправить текст** (`dovechat_send_text`)
- **DoveChat: Отправить файл** (`dovechat_send_file`)
- **DoveChat: Проверить аккаунт** (`dovechat_check_account`)

Регистрируются автоматически при установке приложения. В "Автоматизация продаж" по клику из сделки на "роботы" справа от Роботы и триггеры есть зеленая кнопка +Создать, при нажатии можно найти, выбрав в конце списка категорий в "Другие роботы" в самом низу списка по названию "DoveChat".

### КРИТИЧНО: как триггерить
Робот, поставленный на этап (стадию) сделки, срабатывает **при переходе сделки НА этот этап** — то есть при смене стадии.

### Поля робота "Отправить текст" / "Отправить файл"
- **Телефон клиента** — обязательное
- **ID линии** (`line_id`, UUID) — обязательное
- **Канал** — обязательное, select: Telegram Bot / Telegram User / WhatsApp (Green) / WhatsApp Business API / VK
- **Текст сообщения** — обязательное для send_text, необязательное для send_file
- **Ссылка на файл** (только send_file) — обязательное, должна быть публично доступным URL (например presigned-ссылка MinIO), Битрикс не даёт загрузить файл с диска прямо в форму робота
- **Имя файла** (только send_file) — необязательное

### Поля робота "Проверить аккаунт"
- **Телефон клиента** — обязательное
- **ID линии** — обязательное
- **Канал** — обязательное

### Как увидеть результат "Проверить аккаунт" в карточке сделки
Робот сам пишет результат в кастомное поле сделки **"DoveChat: статус в мессенджере"** (`UF_CRM_DOVECHAT_WA_TG_STATUS`).
Поле нужно один раз добавить в отображение карточки сделки: "Выбрать поле" → найти "DoveChat: статус в мессенджере".
Значения: `Y` — номер зарегистрирован в мессенджере, `N` — не зарегистрирован / ошибка проверки.

## 2. Как сделано программно и интеграционно

### Регистрация роботов
`crm/app/services/crm_service.py :: CrmService._register_robots(self, bx)` вызывается при установке/переустановке приложения (`save_install_tokens()`). Для каждого из трёх `robot_code` сначала удаляет старую регистрацию (`bizproc.robot.delete`, ошибки игнорируются), затем регистрирует заново через `bizproc.robot.add` с `HANDLER=f"{settings.host_url}/crm/api/v1/robots/{robot_code}"`.

Там же создаётся кастомное поле сделки — методом `crm.deal.userfield.add`:
```python
await bx.call("crm.deal.userfield.add", {
    "fields": {
        "FIELD_NAME": "DOVECHAT_WA_TG_STATUS",
        "USER_TYPE_ID": "string",
        "XML_ID": "DOVECHAT_WA_TG_STATUS",
        "EDIT_FORM_LABEL": "DoveChat: статус в мессенджере",
        "LIST_COLUMN_LABEL": "DoveChat: статус",
        "MANDATORY": "N",
    }
}, use_json=True)
```
Битрикс сам добавляет к коду поля префикс — итоговый код `UF_CRM_DOVECHAT_WA_TG_STATUS`, именно его и нужно использовать во всех дальнейших вызовах.

### Приём вебхука
`crm/app/api/v1/robots.py` — `POST /robots/{robot_code}`. Парсит form-data и передаёт в `crm_service.handle_robot(robot_code, data)`.

### `CrmService.handle_robot(robot_code, data)`
Достаёт из payload `event_token`, `member_id`, `phone`, `line_id`, `channel_type`. Находит `CrmAccount` по `member_id`, рефрешит токены. Собирает `bx = BitrixClient(...)`, беря `access_token` из `auth[access_token]` — это рабочий токен для REST-вызовов.

Дальше ветвление:
- `dovechat_check_account` → `_robot_check_account(phone, line_id, channel_type)`, результат сразу пишется в сделку.
- `send_text`/`send_file` → `_robot_send(robot_code, data, conn, line_id, phone, channel_type)`.

В конце — `bizproc.event.send` с `event_token` и `RETURN_VALUES`, чтобы Bitrix понял, что робот завершился.

### Запись результата check_account в сделку
```python
doc_id_raw = data.get("document_id[2]") or data.get("DOCUMENT_ID[2]")  # 'DEAL_17'
if doc_id_raw and doc_id_raw.startswith("DEAL_"):
    deal_id = doc_id_raw.replace("DEAL_", "")
    await bx.call("crm.deal.update", {
        "id": deal_id,
        "fields": {"UF_CRM_DOVECHAT_WA_TG_STATUS": return_values.get("exists")},
    }, use_json=True)
```
`use_json=True` обязателен — без него вложенный `fields` рассыпается на form-encoded передаче.

### `_robot_check_account(phone, line_id, channel_type)`
Явно ветвится по каналу, переданному из формы робота:
- `telegram_user`: берёт `session_string` из `ChannelAccount`, стучится в отдельный сервис **tg-connector** (`http://tg-connector.dchat.svc.cluster.local:8000/prod/check-phone`), получает `{"exists": bool, "user_id": int}`.
- `whatsapp_green`: расшифровывает креды `ChannelAccount`, зовёт Green API `checkWhatsapp`.
- любой другой канал (`telegram_bot`/`waba`/`vk`) — возвращает ошибку "Неподдерживаемый канал".

### `_robot_send(robot_code, data, conn, line_id, phone, channel_type)`
Резолвит `CrmLineConnection` (в `CrmAccount` нет `crm_line_id`/`connector_id`/`channel_type`). Для `send_file` скачивает файл по URL и сохраняет через `StorageService`. Отправляет POST на `chat.dchat.svc.cluster.local:8001/v1/internal/crm-robot-send`.

### `chat_service.py :: ChatService.robot_send(data, tg_listener)`
Ищет или создаёт `Conversation` по `line_id` + `client_phone` + `channel_type` (фильтр по каналу нужен, чтобы не подхватить чужую конверсацию с тем же телефоном от другого канала). Резолвит `ChannelAccount`, создаёт `Message`, инициализирует сендер через `self._init_sender(channel_type, conv.id, tg_listener)` — так `TelegramUserSender` получает `listener` и может отправлять.

`tg_listener` берётся из `app.state.listener` и пробрасывается через `Request` в роутере `chat.py`:
```python
@router.post("/internal/crm-robot-send")
async def crm_robot_send(request: Request, data: CrmMessageOut, chat_service: ChatService = Depends(get_chat_service)):
    return await chat_service.robot_send(data, tg_listener=request.app.state.listener)
```