# infrastructure

соглашения по инфраструктуре leadar.  
всё запускается через docker compose, конфиги в репо `infrastructure`.

---

## сервисы и порты

| сервис | внутренний порт | внешний порт | описание |
|---|---|---|---|
| `nginx` | 80/443 | 80/443 | reverse proxy |
| `backend` | 8000 | — | только через nginx |
| `frontend` | — | — | статика через nginx |
| `postgres` | 5432 | 5432 | только в dev |
| `rabbitmq` | 5672 | — | AMQP |
| `rabbitmq` | 15672 | 15672 | management UI, только в dev |
| `redis` | 6379 | — | если понадобится |

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

    nginx -->|/api/v1| backend
    nginx -->|static| frontend_files["frontend\n(static files)"]
    backend --> postgres
    backend --> rabbitmq
    parser_kwork --> rabbitmq
    parser_fl --> rabbitmq
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

  backend:
    image: ghcr.io/leadar-dev/backend:${VERSION:-latest}
    environment:
      DATABASE__URL: postgresql://leadar:${POSTGRES_PASSWORD}@postgres/leadar_backend
      BROKER__URL: amqp://leadar:${RABBITMQ_PASSWORD}@rabbitmq/leadar
      LOGGING__LEVEL: ${LOGGING_LEVEL:-INFO}
    networks:
      - leadar-network
    depends_on:
      - postgres
      - rabbitmq
    restart: unless-stopped

  parser-kwork:
    image: ghcr.io/leadar-dev/parser-kwork:${VERSION:-latest}
    environment:
      RABBITMQ__URL: amqp://leadar:${RABBITMQ_PASSWORD}@rabbitmq/leadar
      DATABASE__URL: postgresql+asyncpg://leadar:${POSTGRES_PASSWORD}@postgres/leadar_backend
      KWORK__COOKIES: ${KWORK_COOKIES}
    networks:
      - leadar-network
    depends_on:
      - rabbitmq
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
  *.dead                  dead letter для каждой очереди
```

`definitions.json` задаёт эту конфигурацию декларативно — rabbitmq подхватывает при старте.

---

## nginx — routing

```nginx
# nginx/conf.d/backend.conf

upstream backend {
    server backend:8000;
}

server {
    listen 80;
    server_name leadar.local;

    # REST API
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # frontend static
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
KWORK_COOKIES=phpsessid=abc; kwtoken=xyz
LOGGING_LEVEL=INFO
VERSION=latest
```

`.env.example` коммитим с пустыми секретами.