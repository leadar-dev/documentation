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
| `frontend` | vanilla js | фид заказов, фильтры, графики |
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

    subgraph core["Core"]
        BE[backend\nrust / axum]
        DB[(PostgreSQL\nleadar_backend)]
    end

    subgraph clients["Clients"]
        FE[frontend\nvanilla js]
        BOT[telegram-bot\naiogram]
        DB2[(PostgreSQL\nleadar_bot)]
    end

    PK -->|parser.kwork.want| RMQ
    PF -->|parser.fl.want| RMQ
    PU -->|parser.upwork.want| RMQ

    RMQ -->|backend.wants| BE
    BE <--> DB

    FE -->|REST /api/v1| BE
    BOT -->|REST /api/v1| BE
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
    FE->>BE: GET /api/v1/wants?source=kwork&category_id=38
    BE->>DB: SELECT wants WHERE ...
    DB-->>BE: rows
    BE-->>FE: { ok: true, data: [...], pagination: {...} }
    FE->>FE: рендер карточек

    User->>FE: открыл аналитику
    FE->>BE: GET /api/v1/analytics/zscore?category_id=38
    BE->>DB: SELECT + z-score вычисление
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

## правила межсервисного взаимодействия

- **парсеры → backend** — только через RabbitMQ, никаких прямых HTTP вызовов
- **frontend → backend** — только REST `/api/v1`
- **bot → backend** — только REST `/api/v1`
- **сервисы не ходят в БД друг друга** — у каждого своя база (см. `DATABASE.md`)
- **формат событий** — строго по схеме из `API_CONTRACTS.md`