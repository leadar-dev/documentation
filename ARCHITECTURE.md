# architecture

общая архитектура leadar — микросервисный сканер фриланс площадок.

---

## репозитории

| репо | язык | назначение |
|---|---|---|
| `backend` | rust / axum | REST API, аналитика, приём событий из брокера |
| `parser-kwork` | python / httpx | парсинг kwork.ru, публикация в rabbitmq |
| `parser-fl` | python / httpx | парсинг fl.ru, публикация в rabbitmq |
| `parser-upwork` | python / httpx | парсинг upwork.com, публикация в rabbitmq |
| `telegram-bot` | python / aiogram | уведомления, аналитика для пользователя |
| `frontend` | Svelte 5 / TypeScript / Vite | фид заказов, фильтры, графики |
| `infrastructure` | docker / nginx | compose файлы, конфиги, миграции |
| `docs` | markdown | вся документация проекта |

---

## диаграмма сервисов

```mermaid
graph TB
    subgraph parsers["Parsers"]
        PK[parser-kwork]
        PF[parser-fl]
        PU[parser-upwork]
    end

    subgraph broker["Message Broker"]
        RMQ[(RabbitMQ\nleadar.events)]
    end

    subgraph cache["Cache"]
        DF[(Dragonfly)]
    end

    subgraph core["Core"]
        BE[backend\nrust / axum]
        DB[(PostgreSQL\nleadar_backend)]
    end

    subgraph clients["Clients"]
        FE[frontend\nSvelte 5]
        BOT[telegram-bot\naiogram]
        DB2[(PostgreSQL\nleadar_bot)]
    end

    PK -->|check seen| DF
    PF -->|check seen| DF
    PU -->|check seen| DF

    PK -->|parser.kwork.want| RMQ
    PF -->|parser.fl.want| RMQ
    PU -->|parser.upwork.want| RMQ

    RMQ -->|backend.wants| BE
    BE <--> DB

    BE -->|backend.want.new| RMQ
    RMQ -->|bot.notifications| BOT

    FE -->|REST| BE
    BOT -->|REST| BE
    BOT <--> DB2
```

---

## поток данных — новый заказ

```mermaid
sequenceDiagram
    participant Site as kwork.ru
    participant Parser as parser-kwork
    participant RMQ as RabbitMQ
    participant Backend as backend
    participant DB as PostgreSQL
    participant Bot as telegram-bot
    participant User as Пользователь

    loop каждые N секунд
        Parser->>Site: GET /projects (SSR)
        Site-->>Parser: HTML с window.stateData
        Parser->>Parser: извлечь wants из stateData
        Parser->>RMQ: publish parser.kwork.want
    end

    RMQ->>Backend: consume backend.wants
    Backend->>Backend: нормализация, z-score
    Backend->>DB: upsert want
    Backend->>RMQ: publish backend.want.new (если новый)
    RMQ->>Bot: consume bot.notifications
    Bot->>User: уведомление в Telegram
```

---

## поток данных — запрос аналитики

```mermaid
sequenceDiagram
    participant User as Пользователь
    participant FE as frontend
    participant BE as backend
    participant DB as PostgreSQL

    User->>FE: открыл дашборд
    FE->>BE: GET /wants?source=kwork&category_id=38
    BE->>DB: SELECT wants WHERE ...
    DB-->>BE: rows
    BE-->>FE: { ok: true, data: [...], pagination: {...} }
    FE->>FE: рендер карточек

    User->>FE: открыл аналитику
    FE->>BE: GET /analytics/zscore?category_id=38
    BE->>DB: SELECT FROM want_scores (precomputed)
    DB-->>BE: rows
    BE-->>FE: { ok: true, data: { scores: [...] } }
    FE->>FE: рендер графика
```

---

## внутренняя структура сервиса (пример — backend)

```mermaid
graph LR
    subgraph backend
        Router --> Handlers
        Handlers --> Services
        Services --> DB[db queries\nsqlx]
        Services --> Broker[broker\nconsumer/publisher]
        DB --> PG[(PostgreSQL)]
        Broker --> RMQ[(RabbitMQ)]
    end
```

---

## домены

| окружение | frontend | backend API |
|---|---|---|
| prod | `leadar.qu1nqqy.ru` | `api.leadar.qu1nqqy.ru` |
| dev | `dev.leadar.qu1nqqy.ru` | `api.dev.leadar.qu1nqqy.ru` |

API версионирования нет. Разделение — через сабдомен, не префикс `/api/v1`.

---

## авторизация

Проект **приватный**. Доступ — по whitelist telegram_id.

- **telegram-bot**: middleware проверяет `message.from.id` против `ALLOWED_TELEGRAM_IDS` (env var, список через запятую). Неизвестный ID — отвечаем шаблоном: _"Leadar пока что работает в закрытом режиме. Доступ по подписке — скоро."_
- **frontend**: авторизация через Telegram Login Widget → backend выдаёт httpOnly JWT cookie. Не JWT в body.
- **backend**: все эндпоинты защищены JWT middleware. Исключение — `/health`.

В будущем: отдельный платёжный микросервис, расширение модели доступа.

---

## правила межсервисного взаимодействия

- **парсеры → backend** — только через RabbitMQ, никаких прямых HTTP вызовов
- **парсеры → Dragonfly** — только для проверки дедупликации, не для хранения данных
- **frontend → backend** — только REST через `api.leadar.qu1nqqy.ru`
- **bot → backend** — только REST (синхронные запросы по команде юзера)
- **backend → bot** — только через RabbitMQ (`backend.want.new` → `bot.notifications`)
- **сервисы не ходят в БД друг друга** — у каждого своя база (см. `DATABASE.md`)
- **формат событий** — строго по схеме из `API_CONTRACTS.md`