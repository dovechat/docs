### Временная реализации привязки канала VK в личном кабинете.

## Кнопка "Подключить VK"
/opt/dovechat/fe_main/src/app/lc/VkOAuthConnect.jsx — весь файл, кнопка в return <Button onClick={handleConnect}>
## Backend endpoint старта OAuth
/opt/dovechat/mono/account/app/api/v1/vk_oauth_router.py, функция vk_oauth_start (роут GET /oauth/start)
## state/code_verifier/code_challenge
/opt/dovechat/mono/account/app/service/vk_oauth_service.py, метод build_authorize_url. Смысл на русском: это защита от подмены — при старте генерится случайная строка (code_verifier, храним у себя, зашифровав в state), а VK получает только её хэш (code_challenge). При обмене кода на токен мы присылаем оригинал code_verifier, VK сверяет хэш — так подтверждается, что токен запрашивает тот же, кто начал вход, а не перехватчик ссылки.
## Callback endpoint
/opt/dovechat/mono/account/app/api/v1/vk_oauth_router.py, функция vk_oauth_callback (роут GET /oauth/callback)
## Обмен code на token
/opt/dovechat/mono/account/app/service/vk_oauth_service.py, метод handle_callback, блок token_resp = await client.post(VK_TOKEN_URL...)
## Сохранение в credentials_encrypted
/opt/dovechat/mono/account/app/service/vk_oauth_service.py, метод confirm_group, строка encrypted = encrypt(json.dumps(creds_dict))
Функция encrypt — /opt/dchat_prod/infra/crypto.py
## Экран выбора сообщества
/opt/dovechat/fe_main/src/app/lc/VkOAuthConnect.jsx, блок if (groups) { ... }
## Обработка ошибок
/opt/dovechat/mono/account/app/api/v1/vk_oauth_router.pyy, функция vk_oauth_callback, ветка if error or not code or not state
/opt/dovechat/mono/account/app/service/vk_oauth_service.py, handle_callback — проверка state_data["exp"], confirm_group — проверка data["exp"]


## Часть 1 — работает прямо сейчас, без VK-приложения:
Пункты 1, 2, 3, 4, 6, 7 (код), 8 (частично) — вся архитектура и код готовы, запускаются, просто дальше пункта 4 реальный флоу не доходит, потому что VK блокирует само приложение.
## Часть 2 — заработает только после верификации VK-приложения (юрлицо пройдёт проверку):
Пункт 5 (обмен code→token) — не проверен вживую, упадёт пока приложение не активно.
Пункт 7 (список групп реально придёт) — зависит от пункта 5.
Пункт 8 (ошибки внутри handle_callback) — не протестированы на реальных ответах VK, могут давать 500 вместо аккуратной обработки.