# DoveChat — Инструкция по развёртыванию с нуля

## 1. Обзор стека

| Компонент | Технология |
|---|---|
| Оркестрация | k3s (Kubernetes) |
| Контейнеры | Docker + локальный Registry |
| Бэкенд | Python 3.12, FastAPI, SQLAlchemy async |
| БД | PostgreSQL 16 |
| Кэш / брокер | Redis |
| Очереди | Kafka |
| Объектное хранилище | MinIO |
| Планировщик | Airflow 3.0 |
| Фронтенд | React 18 / Vite |
| Reverse proxy | Nginx (Docker-контейнер nginx-1, порт 8443 HTTPS) |

### Сервисы и NodePort-порты (namespace: dchat)

| Сервис | NodePort |
|---|---|
| account-service | 30082 (внутренний порт 8071) |
| chat-service | 30084 |
| crm-service | 30088 |
| admin-service | 30085 |
| telegram-consumer-bot | 30083 |
| fe_main | 30087 |
| frontend-admin | 30089 |
| MinIO | 31001 |
| Grafana | 31005 |
| Airflow | 31006 |

---

## 2. Требования к серверу

**Минимум:** 4 CPU, 8 ГБ RAM, 60 ГБ SSD, Ubuntu 22.04 LTS, внешний IP.  
**Рекомендуется:** 8 CPU, 16 ГБ RAM, 120+ ГБ SSD.

Открытые порты: `80`, `443`, `8443` (HTTPS DoveChat), `22` (SSH).

---

## 3. Установка базового ПО

### Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

### k3s (без Traefik — он конфликтует с Nginx)

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
sudo kubectl get nodes
```

### Локальный Docker Registry

```bash
docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

Разрешить insecure registry для k3s — создать `/etc/rancher/k3s/registries.yaml`:

```yaml
mirrors:
  "localhost:5000":
    endpoint:
      - "http://localhost:5000"
```

```bash
sudo systemctl restart k3s
```

---

## 4. Структура файлов на сервере

```
/opt/dovechat/
├── mono/                  # монорепозиторий (весь бэкенд)
├── fe_main/               # фронтенд (основной + lk)
├── frontend_admin/        # фронтенд admin-панели
│   └── dist/              # собранный дистрибутив (hostPath → под)
├── kuber/
│   └── manifests/         # манифесты Kubernetes
└── nginx/
    └── nginx.conf
```

### Клонирование

```bash
sudo mkdir -p /opt/dovechat
sudo chown $(id -u):$(id -g) /opt/dovechat
cd /opt/dovechat
git clone https://{token}@ithub.com/dovechat/dchat.git mono
git clone https://{token}@github.com/dovechat/fe-main.git fe_main
git clone https://{token}@github.com/dovechat/tg_connector.git
git clone https://{token}@github.com/dovechat/fe-admin.git fe_admin
git clone https://{token}@github.com/dovechat/tg_connector.git
git clone https://{token}@github.com/dovechat/minio.git
```

---

## 5. Kubernetes namespace и секреты

### Namespace

```bash
sudo kubectl create namespace dchat
```

### Секрет приложения

Заполнить под своё окружение:

```bash
kubectl apply -f /opt/dovechat/kuber/secrets/dchat-env-secret.yaml

## пока пропущено
Сгенерировать Fernet key:
```bash
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```
и обновить секрет:
```bash
kubectl apply -f /opt/dovechat/kuber/secrets/dchat-env-secret.yaml
```

---

## 6. База данных

### Первый запуск — применить манифест и дождаться старта

```bash
sudo kubectl apply -f /opt/dovechat/kuber/manifests/db.yaml -n dchat
sudo kubectl wait --for=condition=Ready pod -l app=db -n dchat --timeout=120s
```

### На новой машине — восстановить из дампа

Дамп снимается с работающего инстанса:

```bash
sudo kubectl exec -n dchat deployment/db -- \
  pg_dump -U <user> dovechat > /opt/dovechat/backup/dovechat_$(date +%Y%m%d).sql
```

Восстановление на новой машине (после старта пода):

```bash
# Скопировать дамп в под:
sudo kubectl cp /opt/dovechat/backup/dovechat_YYYYMMDD.sql \
  dchat/<postgres-pod-name>:/tmp/dump.sql

# Восстановить:
sudo kubectl exec -n dchat <postgres-pod-name> -- \
  psql -U <user> -d dovechat -f /tmp/dump.sql
```

Если дампа нет — применить миграции Alembic (только для чистой БД):

```bash
sudo kubectl exec -n dchat deployment/account -- \
  sh -c "cd ../ && alembic upgrade head"
```

---

## 7. Инфраструктурные сервисы

```bash
sudo kubectl apply -f /opt/dovechat/kuber/manifests/redis.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/zookeeper.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/kafka.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/minio.yaml -n dchat
```

Проверка:

```bash
sudo kubectl get pods -n dchat
```

### Создать бакет MinIO (только при первом запуске)

```bash
sudo kubectl exec -n dchat deployment/minio -- \
  mc alias set local http://localhost:9000 <access_key> <secret_key>
sudo kubectl exec -n dchat deployment/minio -- \
  mc mb local/dovechat-media
sudo kubectl exec -n dchat deployment/minio -- \
  mc anonymous set download local/dovechat-media
```

---

## 8. Сборка и публикация Docker-образов

Контекст сборки всегда `/opt/dovechat/mono/` — сервисы импортируют из корня монорепы.

### account-service

```bash
docker ps -a --filter ancestor=localhost:5000/account:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/account:1; docker build --no-cache -t localhost:5000/account:1 -f /opt/dovechat/mono/account/Dockerfile /opt/dovechat/mono && docker push localhost:5000/account:1 && kubectl delete pod -n dchat -l app=account --grace-period=0 --force
```

### chat-service

```bash
docker ps -a --filter ancestor=localhost:5000/chat:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/chat:1; docker build --no-cache -t localhost:5000/chat:1 -f /opt/dovechat/mono/chat/Dockerfile /opt/dovechat/mono && docker push localhost:5000/chat:1 && kubectl delete pod -n dchat -l app=chat --grace-period=0 --force
```

### crm-service

```bash
docker ps -a --filter ancestor=localhost:5000/crm:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/crm:1; docker build --no-cache -t localhost:5000/crm:1 -f /opt/dovechat/mono/crm/Dockerfile /opt/dovechat/mono && docker push localhost:5000/crm:1 && kubectl delete pod -n dchat -l app=crm --grace-period=0 --force
```

### admin-service

```bash
docker ps -a --filter ancestor=localhost:5000/admin:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/admin:1; docker build --no-cache -t localhost:5000/admin:1 -f /opt/dovechat/mono/admin/Dockerfile /opt/dovechat/mono && docker push localhost:5000/admin:1 && kubectl delete pod -n dchat -l app=admin --grace-period=0 --force
```


### fe_main

```bash
docker ps -a --filter ancestor=localhost:5000/fe_main:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/fe_main:1; docker build --no-cache -t localhost:5000/fe_main:1 /opt/dovechat/fe_main/ && docker run --rm -v /opt/dovechat/fe_main/dist:/output localhost:5000/fe_main:1 sh -c "cp -r /app/dist/. /output/ && chmod -R 777 /output" && docker push localhost:5000/fe_main:1 && kubectl delete pod -n dchat -l app=fe-main --grace-period=0 --force
```

### fe_admin

```bash
docker ps -a --filter ancestor=localhost:5000/fe-admin:1 -q | xargs -r docker rm -f && docker rmi localhost:5000/fe-admin:1; docker build --no-cache -t localhost:5000/fe-admin:1 -f /opt/dovechat/fe_admin/Dockerfile /opt/dovechat/fe_admin && docker push localhost:5000/fe-admin:1 && kubectl delete pod -n dchat -l app=fe-admin --grace-period=0 --force
```

> `frontend_admin` использует hostPath: dist монтируется из `/opt/dovechat/frontend_admin/dist` в под, `serve.json` лежит там же.

---

## 9. Применение манифестов сервисов

```bash
sudo kubectl apply -f /opt/dovechat/kuber/manifests/namespace.yaml
sudo kubectl apply -f /opt/dovechat/kuber/manifests/account.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/chat.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/crm.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/admin.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/telegram-consumer-bot.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/telegram-consumer-user.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/fe_main.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/frontend_admin.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/airflow.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/prometheus.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/grafana.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/kube-state-metrics.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/node-exporter.yaml -n dchat
sudo kubectl apply -f /opt/dovechat/kuber/manifests/cadvisor.yaml -n dchat
```

> В манифестах у всех бэкенд-сервисов должен быть `imagePullPolicy: Always` — иначе k3s не подтянет свежий образ с тем же тегом `:1`.

Проверка:

```bash
sudo kubectl get pods -n dchat
# Все поды должны быть Running
```

---

## 10. Nginx

Nginx запущен как Docker-контейнер `nginx-1` из-под docker compose (вне кубера), слушает порт `443` (HTTPS), внешний к k3s, находится в /opt/dovechat/nginx

### Перезапуск Nginx

```bash
cd /opt/dovechat/nginx
docker compose down && docker compose up -d 
```

---

## 11. Airflow 3.0

до первого запуска надо сходить в консоль базы и создать та DB airflow
CREATE DATABASE airflow;
CREATE USER airflow_user WITH PASSWORD '7v3j0sd26vGh@db';
GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow_user;

-- эту команду надо выполнить, перейдя в базу airflow 
GRANT ALL ON SCHEMA public TO airflow_user;

```bash
sudo kubectl exec -n dchat deployment/airflow -- airflow dags reserialize
```

---

## 12. Обновление сервиса

Стандартный цикл (пример для crm):

1. Собрать и запушить новый образ (см. раздел 8).
2. Пересоздать под:

```bash
sudo kubectl delete pod -n dchat -l app=crm --grace-period=0 --force
```

3. Проверить что под поднялся:

```bash
sudo kubectl get pods -n dchat -l app=crm
```

4. Проверить логи:

```bash
sudo kubectl logs -n dchat deployment/crm --tail=50
```

---

## 13. Диагностика

### Общие команды

```bash
# Состояние всех подов:
sudo kubectl get pods -n dchat

# Describe при ошибке:
sudo kubectl describe pod -n dchat <pod-name>

# Войти в под:
sudo kubectl exec -it -n dchat deployment/<service> -- bash

# Логи:
sudo kubectl logs -n dchat deployment/<service> --tail=100 -f
```

### Проверить значение секрета

```bash
sudo kubectl get secret dovechat-secret -n dchat \
  -o jsonpath="{.data.DATABASE_URL}" | base64 --decode
```


*DoveChat © 2026*