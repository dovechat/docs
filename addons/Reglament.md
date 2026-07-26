# Регламент эксплуатации DoveChat

## 1. Резервное копирование

**Что бэкапится:** PostgreSQL (БД), медиафайлы (MinIO).

**БД:**
- Дамп снимается `pg_dump` в plain SQL формате, сжимается gzip
- Хранится в MinIO
- Файл: `dump_{дата}.sql.gz`

**Медиа:**
- Инкрементальный бэкап из MinIO в hostPath `/opt/dovechat/media_backup/YYYY-MM-DD/`

**Хранение:** 365 дней, затем автоматическая очистка.

**Задачи DAG:** `dump_db` → `upload_dump` → `cleanup_dump` → `backup_media` → `cleanup_media`.

## 2. Восстановление

**БД:**
```bash
gunzip -c dump_YYYY-MM-DD.sql.gz | psql $(echo $DATABASE_URL | sed 's|postgresql+asyncpg://|postgresql://|')
```

**Медиа:**
```bash
mc mirror /opt/dovechat/media_backup/YYYY-MM-DD/ <alias>/<bucket>/
```

## 3. Мониторинг и алерты

Мониторинг метрик — Prometheus + Grafana.

Контроль выполнения бэкапов — через встроенный интерфейс Airflow (статусы тасков DAG: `dump_db`, `upload_dump`, `cleanup_dump`, `backup_media`, `cleanup_media` — успех/ошибка по каждому запуску, история запусков, логи по каждой задаче).

### Подробно по пунктам
#### Kafka lag

Раздел: Explore
Ссылка: /grafana/explore → datasource prometheus
Название: kafka_consumergroup_lag
Суть: отставание consumer-групп по offset'ам в очередях
Если пусто: дёрни нагрузку на любой топик или подожди — экспортер сам пушит постоянно, пусто не должно быть в принципе; если пусто — глянь Targets, up ли kafka-exporter

#### Pod failed (алерт)

Раздел: Alerting → Alert rules
Ссылка: /grafana/alerting/grafana/efpujiwnlrdvkc/view
Название: Pod failed
Суть: срабатывает если под в dchat не в Ready
Если пусто/не триггерится: всё ок, значит поды живы — специально ронять под не нужно

#### 500 in logs (алерт)

Раздел: Alerting → Alert rules
Ссылка: /grafana/alerting/grafana/ffpukb7cziby8d/view
Название: 500 in logs
Суть: считает строки level=error в логах за 5 минут (через Loki)
Если пусто: вызови любой 500-эндпоинт, глянь через Explore datasource loki

#### Kafka consumer lag > 2000 (алерт, сегодня добавлен)

Раздел: Alerting → Alert rules
Ссылка: https://dev.dove-chat.pro/grafana/alerting/grafana/ffsu69afwt98gc/view
Название: Kafka consumer lag (как назвал)
Суть: тот же kafka_consumergroup_lag, но с порогом и алертом
Если пусто: то же, что и п.1


#### Webhook processing time
Раздел: Explore
Ссылка: /grafana/explore → datasource prometheus
Название: dovechat_webhook_processing_seconds_count (и _sum, _bucket)
Суть: время обработки входящих вебхуков (TG bot, whatsapp_green, waba)
Если пусто: отправить тестовое сообщение 
количество - https://dev.dove-chat.pro/grafana/d/adzxm6d/main?viewPanel=panel-10
время обработки - https://dev.dove-chat.pro/grafana/d/adzxm6d/main?viewPanel=panel-11


#### Webhook requests (success/error)
Раздел: Grafana Dashboard main
Ссылка: добавь панель, дам прямую после сохранения — сейчас создай сам через Add Panel
Название: dovechat_webhook_requests_total
Суть: счётчик вызовов вебхука по статусу success/error (TG bot; Green/WABA — после проверки)
Если пусто: отправить тестовое сообщение / вызвать реальную ошибку в обработке
https://dev.dove-chat.pro/grafana/d/adzxm6d/main?viewPanel=panel-12

#### Webhook error rate (%)
Раздел: Grafana Dashboard main
Ссылка: добавь панель, дам прямую после сохранения
Название: rate(dovechat_webhook_requests_total{status="error"}[5m]) / rate(dovechat_webhook_requests_total[5m]) * 100
Суть: процент ошибок среди всех вызовов вебхука за последние 5 минут
Если пусто: нужны и success, и error события в окне 5 минут одновременно
https://dev.dove-chat.pro/grafana/d/adzxm6d/main?viewPanel=panel-13


## 4. Управление инстансами

**Kubernetes (k3s):** Проект состоит из множества независимых сервисов (chat, crm, account, admin и фронтенды, kafka, apache airflow, telegram connector, db, redis), которые нужно обновлять, масштабировать и перезапускать по отдельности, не затрагивая остальные. Kubernetes даёт для этого готовый механизм — изолированные под-контейнеры на сервис, единообразный способ раскатки новых версий, автоматический перезапуск упавших подов и возможность увеличить число реплик конкретного сервиса при росте нагрузки, не трогая архитектуру целиком. 

**Топология:** k3s, namespace `dchat`, разворачивается на VPS 147.45.254.165 (Москва).

**Секреты и конфиги:** переменные окружения сервисов (например `AMO_CLIENT_ID`, `DATABASE_URL`) заданы в спецификациях секретах, хранятся в /kuber/secrets. Посмотреть:
```bash
kubectl get deploy <service> -n dchat -o yaml
```
Изменить — правка манифеста → применение → перезапуск, чтобы под подхватил новые переменные:
```bash
kubectl rollout restart deploy/<service> -n dchat
```

**Сеть/выход наружу:** часть сервисов имеют NodePort (см. приложение А) — доступны напрямую по `147.45.254.165:<порт>`. Для сервисов без NodePort — временный проброс:
```bash
kubectl port-forward --address 0.0.0.0 -n dchat deploy/<service> <порт>:<containerPort>
```

**Nginx** отдельный сервер вне k3s-кластера. Через него идёт маршрутизация внешних запросов по путям — не по доменным именам, а по `location`-префиксам на нужный внутренний сервис: `/crm/` → порт 30088 (crm), `/mproxy/` → приём вебхуков TG/WhatsApp, `/dev/` → туннель на рабочую машину разработчика. Nginx здесь — единая точка входа снаружи, разводящая трафик по бэкендам DoveChat в зависимости от префикса URL.

**Просмотр состояния кластера:**
```bash
kubectl get pods -n dchat
kubectl get deploy -n dchat
kubectl logs -n dchat deploy/<service> --tail=100 -f
```

**Масштабирование:**
```bash
kubectl scale deploy/<service> -n dchat --replicas=<N>
```

### Приложение А. Известные NodePort'ы

| Сервис | NodePort | Путь в nginx |
|---|---|---|
| account | 30082 | `/account/` |
| admin (backend) | 30085 | `/admin/` |
| crm | 30088 | `/crm/` |
| frontend_admin (статика) | 30089 | `/panel/` |
| fe_main (chat/embed) | 30087 | — |

## 5. Действия при сбоях каналов

При сбое отдельного канала (Telegram/WhatsApp/VK) на стороне DoveChat: проверка состояния подключения через листенер канала. Для Telegram есть автоматическое переподключение (`TelegramListener._health_check_loop`, проверка раз в 120 секунд). Для WhatsApp Green API есть отслеживание обрыва соединения с уведомлением при потере связи. Если автоматика не восстановила канал — ручной рестарт соответствующего пода.

Если недоступность вызвана блокировкой/сетевыми ограничениями (не сбоем самого канала) — используется резервный прокси-сервер (сейчас — бразильский `tg.dove-chat.pro`), желаетельно добавить дополнительные прокси.

## 6. Инциденты