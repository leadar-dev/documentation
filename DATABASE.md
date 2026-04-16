# database conventions

соглашения по базам данных для сервисов leadar.  
**каждый сервис — своя база. прямой доступ к чужой БД запрещён.**

---

## изоляция баз

| сервис | база | владелец данных |
|---|---|---|
| `backend` | `leadar_backend` | wants, categories, аналитика |
| `telegram-bot` | `leadar_bot` | пользователи, подписки, настройки |

парсеры и frontend **не имеют своей БД** — парсеры пишут в брокер, frontend читает через REST.

---

## схема — leadar_backend

```mermaid
erDiagram
    wants {
        bigserial id PK
        varchar source "kwork | fl | upwork"
        bigint external_id "id на площадке"
        varchar name
        text description
        numeric price_limit
        numeric possible_price_limit
        int category_id FK
        int max_days
        varchar status "active | archive | closed"
        int kwork_count "кол-во откликов"
        int views
        numeric hired_percent
        varchar url
        timestamptz date_create
        timestamptz date_expire
        timestamptz parsed_at
        timestamptz created_at
        timestamptz updated_at
    }

    categories {
        serial id PK
        varchar source
        int external_id
        varchar name
        int parent_id FK
        timestamptz created_at
    }

    want_scores {
        bigserial id PK
        bigint want_id FK
        numeric zscore_price
        numeric zscore_activity
        numeric trend_slope
        timestamptz calculated_at
    }

    wants ||--o{ want_scores : "has"
    categories ||--o{ wants : "has"
    categories ||--o{ categories : "parent"
```

### индексы — leadar_backend

```sql
-- основные запросы — фильтрация по source, category, status
CREATE INDEX idx_wants_source ON wants(source);
CREATE INDEX idx_wants_category_id ON wants(category_id);
CREATE INDEX idx_wants_status ON wants(status);
CREATE INDEX idx_wants_date_create ON wants(date_create DESC);

-- upsert по внешнему id
CREATE UNIQUE INDEX idx_wants_source_external ON wants(source, external_id);
```

---

## схема — leadar_bot

```mermaid
erDiagram
    users {
        bigserial id PK
        bigint telegram_id UK
        varchar username
        varchar language "ru | en"
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    subscriptions {
        bigserial id PK
        bigint user_id FK
        varchar source "kwork | fl | upwork | all"
        int category_id "null = все категории"
        numeric price_min
        numeric price_max
        boolean is_active
        timestamptz created_at
    }

    notification_log {
        bigserial id PK
        bigint user_id FK
        bigint want_id "внешний id из backend"
        timestamptz sent_at
    }

    users ||--o{ subscriptions : "has"
    users ||--o{ notification_log : "has"
```

### индексы — leadar_bot

```sql
CREATE UNIQUE INDEX idx_users_telegram_id ON users(telegram_id);
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_active ON subscriptions(is_active) WHERE is_active = true;
CREATE INDEX idx_notification_log_user_want ON notification_log(user_id, want_id);
```

---

## соглашения по схемам

### нейминг

```sql
-- таблицы — snake_case, множественное число
wants, categories, want_scores, notification_log

-- колонки — snake_case
external_id, price_limit, date_create

-- индексы — idx_таблица_колонка
idx_wants_source, idx_users_telegram_id

-- FK — таблица_id
user_id, want_id, category_id
```

### типы колонок

| данные | тип |
|---|---|
| ID | `bigserial` (PK), `bigint` (FK) |
| деньги/цены | `numeric(12, 2)` |
| проценты | `numeric(5, 2)` |
| короткий текст | `varchar(255)` |
| длинный текст | `text` |
| дата+время | `timestamptz` (всегда UTC) |
| булево | `boolean` |
| enum-строки | `varchar` + check constraint |

```sql
-- enum через check
ALTER TABLE wants ADD CONSTRAINT chk_wants_status
    CHECK (status IN ('active', 'archive', 'closed'));

ALTER TABLE wants ADD CONSTRAINT chk_wants_source
    CHECK (source IN ('kwork', 'fl', 'upwork'));
```

### обязательные колонки

каждая таблица имеет:

```sql
created_at  timestamptz NOT NULL DEFAULT now()
updated_at  timestamptz NOT NULL DEFAULT now()
```

для `updated_at` — триггер:

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_wants_updated_at
    BEFORE UPDATE ON wants
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## миграции

используем `sqlx migrate` для backend, `alembic` для bot.

```
backend/
  migrations/
    0001_create_categories.sql
    0002_create_wants.sql
    0003_create_want_scores.sql
    0004_add_wants_indexes.sql

telegram-bot/
  migrations/
    0001_create_users.sql
    0002_create_subscriptions.sql
    0003_create_notification_log.sql
```

### правила миграций

- **только вперёд** — down миграций не пишем
- **одна миграция — одно изменение**
- **не редактируем применённые миграции** — только новая миграция
- имя файла: `NNNN_описание_snake_case.sql`