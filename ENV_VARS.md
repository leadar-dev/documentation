# env vars conventions

соглашения по конфигурации для python-сервисов leadar.  
основной конфиг — `config.toml`. секреты — переменные окружения.

---

## подход

два источника конфигурации, приоритет слева направо:

```
env vars  >  config.toml
```

- **`config.toml`** — не-секретные настройки, коммитим в репо (с дефолтами)
- **env vars** — секреты и оверрайды для прода, не коммитим
- **дефолты в коде** — для локальной разработки без config.toml

`pydantic-settings` собирает всё автоматически через `settings_customise_sources`.

---

## config.toml — структура

```toml
[kwork]
base_url = "https://kwork.ru"
request_delay = 1.5
user_agent = "Mozilla/5.0 ..."

[logging]
level = "INFO"

[rabbitmq]
url = "amqp://guest:guest@localhost/"   # в проде — через env

[database]
url = "postgresql+asyncpg://postgres:postgres@localhost/leadar"  # в проде — через env

[dragonfly]
url = "redis://localhost:6379"          # в проде — через env
dedup_ttl_days = 7                      # TTL ключей дедупликации

[parser]
interval = 60
```

`config.toml` коммитим — он содержит только дефолты для разработки, секретов нет.

---

## env vars — только для секретов и прода

формат оверрайда через `env_nested_delimiter="__"`:

```
СЕКЦИЯ__ПОЛЕ=значение
```

```bash
# секреты — никогда не в toml
RABBITMQ__URL=amqp://user:secret@rabbit.prod/leadar
DATABASE__URL=postgresql+asyncpg://user:secret@db.prod/leadar
DRAGONFLY__URL=redis://dragonfly:6379

# оверрайды для прода
LOGGING__LEVEL=WARNING
PARSER__INTERVAL=30
```

регистр не важен (`case_sensitive=False`) — можно и строчными.

---

## что куда идёт

| параметр | config.toml | env var |
|---|---|---|
| `kwork.base_url` | ✅ дефолт | — |
| `kwork.request_delay` | ✅ дефолт | 🟡 если надо оверрайднуть |
| `rabbitmq.url` | локальный дефолт | ✅ в проде |
| `database.url` | локальный дефолт | ✅ в проде |
| `dragonfly.url` | локальный дефолт | ✅ в проде |
| `dragonfly.dedup_ttl_days` | `7` | 🟡 если надо |
| `logging.level` | `"INFO"` | 🟡 если надо |

---

## нейминг полей

- **секции** — по компоненту: `kwork`, `rabbitmq`, `database`, `parser`, `logging`
- **поля** — `snake_case`, описательно без аббревиатур

```python
# ❌
req_delay: float
lvl: str

# ✅
request_delay: float
level: str
```

---

## добавление новой секции

1. создаём `BaseModel` класс
2. добавляем в `Config`
3. добавляем секцию в `config.toml` с дефолтами
4. секреты — пустая строка + комментарий как переопределить

```python
class Telegram(BaseModel):
    token: str = Field(default="", description="Bot token from @BotFather")
    webhook_url: str = Field(default="")

class Config(BaseSettings):
    ...
    telegram: Telegram = Telegram()
```

```toml
[telegram]
token = ""          # required in prod: TELEGRAM__TOKEN=...
webhook_url = ""    # required in prod: TELEGRAM__WEBHOOK_URL=...
```

---

## gitignore

```gitignore
.env
.env.*
# config.toml НЕ в gitignore — коммитим с дефолтами
```

секреты передаём через env vars напрямую (docker, systemd, CI/CD) — без `.env` файлов в проде.