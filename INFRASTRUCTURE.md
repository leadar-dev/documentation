# infrastructure

соглашения по инфраструктуре leadar.
конфиги в репо `infrastructure`.

---

## prod vs dev — стратегия деплоя

### dev и prod

postgres, rabbitmq, nginx, prometheus — **синглтоны на хосте (systemd)**, shared между всеми проектами на машине.

в docker compose живут только leadar-специфичные сервисы:

```
VM:
  systemd:
    postgresql        — данные на хосте, независимы от docker
    rabbitmq          — очередь не падает при рестарте compose
    nginx             — точка входа, не зависит от состояния compose
    prometheus        — shared мониторинг для всех проектов
    postgres-exporter — shared, scrape в shared prometheus

  docker compose (leadar):
    backend
    parser-kwork (+ другие парсеры)
    telegram-bot
    dragonfly         — кэш, данные ephemeral
```

**почему stateful сервисы вне compose:**
- рестарт docker daemon / compose не роняет БД и брокер
- I/O без overhead overlay2 filesystem
- backup проще: `pg_dump` напрямую, без `docker exec`

**dragonfly в compose** — данные некритичны (дедупликация), пересоздаётся без потерь.

**kubernetes** — не используем. оверхед без выхлопа для этого масштаба. проблема OOM killer актуальна для k8s (pod eviction), в compose — управляется через `mem_limit`.

---

## сервисы и порты

| сервис | порт | где живёт | описание |
|---|---|---|---|
| `nginx` | 80/443 | systemd | reverse proxy |
| `postgres` | 5432 | systemd | БД |
| `rabbitmq` | 5672 | systemd | AMQP |
| `rabbitmq` | 15672 | systemd | management UI |
| `prometheus` | 9090 | systemd | сбор метрик |
| `postgres-exporter` | 9187 | systemd | метрики postgres |
| `backend` | 8000 | compose | только через nginx |
| `frontend` | — | compose | статика через nginx (Svelte 5) |
| `dragonfly` | 6379 | compose | Redis-совместимый кэш, дедупликация парсеров |

---

## структура infrastructure репо

```
infrastructure/
  docker-compose.yml            — все сервисы
  docker-compose.dev.yml        — оверрайды для разработки
  nginx/
    conf.d/
      api.conf                  — проксирование на backend
      frontend.conf             — статика frontend
  postgres/
    init/
      01_init.sh                — создание баз и пользователей при первом запуске
  rabbitmq/
    rabbitmq.conf               — указывает путь к definitions.json
    definitions.json            — exchanges, queues, DLQ конфиг
  prometheus/
    prometheus.yml              — конфиг скрейпинга
```

---

## docker compose

```mermaid
graph TB
    subgraph systemd["systemd (host)"]
        nginx["nginx\n:80/:443"]
        postgres["postgres\n:5432"]
        rabbitmq["rabbitmq\n:5672"]
    end

    subgraph docker-network["leadar-network (bridge)"]
        backend["backend\n:8000"]
        parser_kwork["parser-kwork"]
        parser_fl["parser-fl"]
        bot["telegram-bot"]
        dragonfly["dragonfly\n:6379"]
    end

    nginx -->|api.leadar| backend
    nginx -->|leadar| frontend_files["frontend\n(Svelte 5 static)"]
    backend --> postgres
    backend --> rabbitmq
    backend --> dragonfly
    parser_kwork --> rabbitmq
    parser_kwork --> dragonfly
    parser_fl --> rabbitmq
    parser_fl --> dragonfly
    bot --> rabbitmq
    bot --> postgres
```

### docker-compose.yml — структура

```yaml
# infrastructure/docker-compose.yml

networks:
  leadar-network:
    driver: bridge

volumes:
  dragonfly-data:

services:

  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly:v1.27.1
    volumes:
      - dragonfly-data:/data
    networks:
      - leadar-network
    restart: unless-stopped
    ulimits:
      memlock: -1
    cap_add:
      - SYS_NICE
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  backend:
    image: ghcr.io/leadar-dev/backend:${VERSION:-latest}
    environment:
      DATABASE__URL: postgresql://backend_user:${BACKEND_DB_PASSWORD}@localhost/leadar_backend
      BROKER__URL: amqp://leadar:${RABBITMQ_PASSWORD}@localhost/
      DRAGONFLY__URL: redis://dragonfly:6379
      LOGGING__LEVEL: ${LOGGING_LEVEL:-INFO}
    networks:
      - leadar-network
    depends_on:
      dragonfly:
        condition: service_healthy
    restart: unless-stopped

  parser-kwork:
    image: ghcr.io/leadar-dev/parser-kwork:${VERSION:-latest}
    environment:
      RABBITMQ__URL: amqp://leadar:${RABBITMQ_PASSWORD}@localhost/
      DRAGONFLY__URL: redis://dragonfly:6379
    networks:
      - leadar-network
    depends_on:
      dragonfly:
        condition: service_healthy
    restart: unless-stopped
```

---

## postgres — инициализация баз

```sql
-- infrastructure/postgres/init/01_create_databases.sql
-- выполняется один раз при первом запуске контейнера

CREATE DATABASE leadar_backend;
CREATE DATABASE leadar_bot;

-- отдельные пользователи для изоляции
CREATE USER backend_user WITH PASSWORD 'changeme';
CREATE USER bot_user WITH PASSWORD 'changeme';

GRANT ALL PRIVILEGES ON DATABASE leadar_backend TO backend_user;
GRANT ALL PRIVILEGES ON DATABASE leadar_bot TO bot_user;
```

---

## rabbitmq — exchanges и queues

```
exchange:  leadar.events  (topic, durable)
exchange:  leadar.dead    (direct, durable)  — dead letter

queues:
  backend.wants           routing: parser.*.want
  bot.notifications       routing: backend.want.new
  backend.wants.dead      dead letter для backend.wants
  bot.notifications.dead  dead letter для bot.notifications
```

`definitions.json` задаёт эту конфигурацию декларативно — rabbitmq подхватывает при старте.

### dead letter queue — обработка

сообщения попадают в DLQ после исчерпания retry или ручного nack без requeue.

backend содержит отдельный consumer DLQ:
- читает `backend.wants.dead` и `bot.notifications.dead`
- логирует с уровнем `error` + полный payload
- инкрементирует Prometheus counter `dlq_messages_total{queue}`

алерт в Prometheus: `dlq_messages_total > 0` → что-то сломалось, требует ручного разбора.

сами сообщения не переотправляем автоматически — только вручную после диагностики.

---

## nginx — routing

два виртуальных хоста: frontend и API разделены сабдоменом.

```nginx
# nginx/conf.d/api.conf

upstream backend {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name api.leadar.qu1nqqy.ru api.dev.leadar.qu1nqqy.ru;

    location /metrics {
        allow 127.0.0.1;
        deny all;
        proxy_pass http://backend;
    }

    location /health {
        proxy_pass http://backend;
    }

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# nginx/conf.d/frontend.conf

server {
    listen 80;
    server_name leadar.qu1nqqy.ru dev.leadar.qu1nqqy.ru;

    location / {
        root /var/www/frontend;
        try_files $uri $uri/ /index.html;
    }
}
```

---

## CI/CD — образы

образы собираем через GitHub Actions и пушим в GitHub Container Registry:

```
ghcr.io/leadar-dev/backend:latest
ghcr.io/leadar-dev/parser-kwork:latest
ghcr.io/leadar-dev/parser-fl:latest
ghcr.io/leadar-dev/parser-upwork:latest
ghcr.io/leadar-dev/telegram-bot:latest
```

тег `latest` — последний стабильный (из `main`).  
тег `dev` — последний из ветки `dev`.

---

## переменные окружения — compose

секреты в compose передаём через `.env` файл рядом с `docker-compose.yml`:

```bash
# infrastructure/.env (в .gitignore)
POSTGRES_USER=leadar
POSTGRES_PASSWORD=
BACKEND_DB_PASSWORD=
BOT_DB_PASSWORD=
RABBITMQ_DEFAULT_USER=leadar
RABBITMQ_DEFAULT_PASS=
DATA_SOURCE_NAME=
LOGGING_LEVEL=INFO
VERSION=latest
```

`.env.example` коммитим с пустыми значениями — документирует какие переменные нужны.