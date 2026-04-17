# infrastructure

соглашения по инфраструктуре leadar.  
всё запускается через docker compose, конфиги в репо `infrastructure`.

---

## сервисы и порты

| сервис | внутренний порт | внешний порт | описание |
|---|---|---|---|
| `nginx` | 80/443 | 80/443 | reverse proxy |
| `backend` | 8000 | — | только через nginx |
| `frontend` | — | — | статика через nginx (Svelte 5) |
| `postgres` | 5432 | 5432 | только в dev |
| `rabbitmq` | 5672 | — | AMQP |
| `rabbitmq` | 15672 | 15672 | management UI, только в dev |
| `dragonfly` | 6379 | — | Redis-совместимый кэш, дедупликация парсеров |
| `prometheus` | 9090 | 9090 | только в dev |
| `postgres-exporter` | 9187 | — | метрики postgres для prometheus |

---

## структура infrastructure репо

```
infrastructure/
  docker/
    docker-compose.yml          — все сервисы
    docker-compose.dev.yml      — оверрайды для разработки
    docker-compose.prod.yml     — оверрайды для прода
  nginx/
    nginx.conf                  — основной конфиг
    conf.d/
      backend.conf              — проксирование /api/v1
      frontend.conf             — статика frontend
  postgres/
    init/
      01_create_databases.sql   — создание баз при первом запуске
  rabbitmq/
    definitions.json            — exchanges, queues, vhosts
```

---

## docker compose

```mermaid
graph TB
    subgraph docker-network["leadar-network (bridge)"]
        nginx["nginx\n:80/:443"]
        backend["backend\n:8000"]
        parser_kwork["parser-kwork"]
        parser_fl["parser-fl"]
        bot["telegram-bot"]
        postgres["postgres\n:5432"]
        rabbitmq["rabbitmq\n:5672\n:15672"]
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
# infrastructure/docker/docker-compose.yml

networks:
  leadar-network:
    driver: bridge

volumes:
  postgres-data:
  rabbitmq-data:
  dragonfly-data:

services:

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: leadar
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d
    networks:
      - leadar-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U leadar"]
      interval: 5s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: leadar
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
      - ./rabbitmq/definitions.json:/etc/rabbitmq/definitions.json
    networks:
      - leadar-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
    volumes:
      - dragonfly-data:/data
    networks:
      - leadar-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  backend:
    image: ghcr.io/leadar-dev/backend:${VERSION:-latest}
    environment:
      DATABASE__URL: postgresql://leadar:${POSTGRES_PASSWORD}@postgres/leadar_backend
      BROKER__URL: amqp://leadar:${RABBITMQ_PASSWORD}@rabbitmq/leadar
      DRAGONFLY__URL: redis://dragonfly:6379
      LOGGING__LEVEL: ${LOGGING_LEVEL:-INFO}
    networks:
      - leadar-network
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      dragonfly:
        condition: service_healthy
    restart: unless-stopped

  parser-kwork:
    image: ghcr.io/leadar-dev/parser-kwork:${VERSION:-latest}
    environment:
      RABBITMQ__URL: amqp://leadar:${RABBITMQ_PASSWORD}@rabbitmq/leadar
      DRAGONFLY__URL: redis://dragonfly:6379
    networks:
      - leadar-network
    depends_on:
      rabbitmq:
        condition: service_healthy
      dragonfly:
        condition: service_healthy
    restart: unless-stopped

  postgres-exporter:
    image: prometheuscom/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: postgresql://leadar:${POSTGRES_PASSWORD}@postgres:5432/leadar_backend?sslmode=disable
    networks:
      - leadar-network
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
    networks:
      - leadar-network
    depends_on:
      - backend
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
    server backend:8000;
}

server {
    listen 80;
    server_name api.leadar.qu1nqqy.ru api.dev.leadar.qu1nqqy.ru;

    # /metrics и /health — только из внутренней сети (Prometheus)
    location /metrics {
        allow 10.0.0.0/8;
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
# infrastructure/docker/.env (в .gitignore)
POSTGRES_PASSWORD=secret
RABBITMQ_PASSWORD=secret
LOGGING_LEVEL=INFO
VERSION=latest
```

`.env.example` коммитим с пустыми значениями — документирует какие переменные нужны.