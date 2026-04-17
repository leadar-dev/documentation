# env vars conventions — rust (backend)

соглашения по конфигурации для backend leadar.
используем `envy` для десериализации переменных окружения в типизированный `Config`.

---

## подход

единственный источник конфигурации — переменные окружения.
никакого `config.toml` в rust-сервисах — только env vars.

```
env vars  →  envy::from_env::<Config>()
```

падаем при старте если обязательная переменная не задана (`fail fast`).

---

## структура Config

```rust
// src/config.rs
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Config {
    pub database: DatabaseConfig,
    pub broker: BrokerConfig,
    pub http: HttpConfig,
    pub auth: AuthConfig,
    pub analytics: AnalyticsConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
}

#[derive(Debug, Deserialize)]
pub struct BrokerConfig {
    pub url: String,
}

#[derive(Debug, Deserialize)]
pub struct HttpConfig {
    #[serde(default = "HttpConfig::default_host")]
    pub host: String,
    #[serde(default = "HttpConfig::default_port")]
    pub port: u16,
}

impl HttpConfig {
    fn default_host() -> String { "0.0.0.0".to_string() }
    fn default_port() -> u16 { 8000 }
}

#[derive(Debug, Deserialize)]
pub struct AuthConfig {
    pub jwt_secret: String,
    pub jwt_expiry_hours: Option<u64>,      // default: 24
    pub telegram_bot_token: String,         // для верификации Telegram Login
    pub allowed_telegram_ids: String,       // "123456,789012" — через запятую
}

#[derive(Debug, Deserialize)]
pub struct AnalyticsConfig {
    #[serde(default = "AnalyticsConfig::default_zscore_interval")]
    pub zscore_interval_minutes: u64,
    #[serde(default = "AnalyticsConfig::default_zscore_window")]
    pub zscore_window_days: u64,
}

impl AnalyticsConfig {
    fn default_zscore_interval() -> u64 { 30 }
    fn default_zscore_window() -> u64 { 7 }
}

#[derive(Debug, Deserialize)]
pub struct LoggingConfig {
    #[serde(default = "LoggingConfig::default_level")]
    pub level: String,
}

impl LoggingConfig {
    fn default_level() -> String { "info".to_string() }
}
```

---

## инициализация в main.rs

```rust
// src/main.rs
use anyhow::Context;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cfg = envy::from_env::<Config>()
        .context("invalid config — check env vars")?;

    // ...
}
```

если переменная обязательная и не задана — `envy` вернёт ошибку, `.context()` добавит сообщение, сервис упадёт сразу с понятным логом.

---

## переменные окружения

формат: `СЕКЦИЯ__ПОЛЕ` (двойное подчёркивание — разделитель вложенности в `envy`).

| переменная | обязательная | дефолт | описание |
|---|---|---|---|
| `DATABASE__URL` | ✅ | — | postgres connection string |
| `BROKER__URL` | ✅ | — | rabbitmq connection string |
| `HTTP__HOST` | — | `0.0.0.0` | адрес прослушивания |
| `HTTP__PORT` | — | `8000` | порт |
| `AUTH__JWT_SECRET` | ✅ | — | секрет подписи JWT (min 32 символа) |
| `AUTH__JWT_EXPIRY_HOURS` | — | `24` | срок жизни токена |
| `AUTH__TELEGRAM_BOT_TOKEN` | ✅ | — | токен бота для верификации login |
| `AUTH__ALLOWED_TELEGRAM_IDS` | ✅ | — | whitelist telegram_id через запятую |
| `ANALYTICS__ZSCORE_INTERVAL_MINUTES` | — | `30` | интервал пересчёта z-score |
| `ANALYTICS__ZSCORE_WINDOW_DAYS` | — | `7` | окно выборки для z-score |
| `LOGGING__LEVEL` | — | `info` | уровень логов (trace/debug/info/warn/error) |

---

## пример .env для docker-compose

```bash
# infrastructure/docker/.env (в .gitignore)
DATABASE__URL=postgresql://leadar:secret@postgres/leadar_backend
BROKER__URL=amqp://leadar:secret@rabbitmq/leadar
AUTH__JWT_SECRET=change_me_min_32_characters_long_secret
AUTH__TELEGRAM_BOT_TOKEN=1234567890:AABBccDDeeFFggHH
AUTH__ALLOWED_TELEGRAM_IDS=123456789,987654321
```

---

## нейминг переменных

- секции: по компоненту (`DATABASE`, `BROKER`, `AUTH`, `ANALYTICS`, `LOGGING`, `HTTP`)
- поля: `UPPER_SNAKE_CASE`
- разделитель вложенности: `__` (два подчёркивания)
