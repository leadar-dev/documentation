# error handling conventions — rust

соглашения по обработке ошибок для rust-сервисов leadar.

---

## инструменты

| где | что | зачем |
|---|---|---|
| библиотечный код, домен | `thiserror` | типизированные ошибки с явными вариантами |
| `main.rs`, точки входа | `anyhow` | быстрый проброс без определения типов |
| axum handlers | `AppError: IntoResponse` | конвертация в HTTP ответ |

---

## AppError — центральный тип ошибок

```rust
// src/errors.rs
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    // доменные
    #[error("want {0} not found")]
    WantNotFound(i64),

    #[error("category {0} not found")]
    CategoryNotFound(i32),

    // инфраструктурные
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("broker error: {0}")]
    Broker(String),

    #[error("invalid payload: {0}")]
    InvalidPayload(String),

    #[error("parse error: {0}")]
    Parse(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::WantNotFound(_)    => (StatusCode::NOT_FOUND, "WANT_NOT_FOUND"),
            AppError::CategoryNotFound(_) => (StatusCode::NOT_FOUND, "CATEGORY_NOT_FOUND"),
            AppError::InvalidPayload(_)  => (StatusCode::BAD_REQUEST, "INVALID_PAYLOAD"),
            _                            => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        // внутренние детали ошибок инфраструктуры — не светим клиенту
        let message = match &self {
            AppError::Database(_) | AppError::Broker(_) => "internal server error".to_string(),
            e => e.to_string(),
        };

        let body = json!({
            "ok": false,
            "error": { "code": code, "message": message }
        });

        (status, Json(body)).into_response()
    }
}
```

---

## ? оператор — основной инструмент

```rust
// ❌ ручной unwrap
let row = sqlx::query_as!(WantRow, "SELECT * FROM wants WHERE id = $1", id)
    .fetch_optional(&pool)
    .await
    .unwrap()
    .unwrap();

// ✅ ? с явной обработкой отсутствия
let row = sqlx::query_as!(WantRow, "SELECT * FROM wants WHERE id = $1", id.0)
    .fetch_optional(&pool)
    .await?                                         // sqlx::Error → AppError::Database
    .ok_or(AppError::WantNotFound(id.0))?;          // None → AppError::WantNotFound
```

---

## context — добавляем информацию к ошибке

используем `.context()` из `anyhow` или ручной `map_err` для добавления контекста:

```rust
use anyhow::Context;

// в main/точке входа — anyhow context
async fn run() -> anyhow::Result<()> {
    let pool = PgPool::connect(&cfg.database.url)
        .await
        .context("failed to connect to database")?;

    Ok(())
}

// в доменном коде — map_err с явным типом
fn parse_price(s: &str) -> AppResult<f64> {
    s.parse::<f64>()
        .map_err(|_| AppError::InvalidPayload(format!("invalid price: {s}")))
}
```

---

## fail fast при старте

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    init_tracing();

    let cfg = Config::from_env().context("invalid config")?;

    let pool = PgPool::connect(&cfg.database.url)
        .await
        .context("database connection failed")?;

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .context("migrations failed")?;

    info!("server starting");
    serve(pool, cfg).await
}
```

если что-то из этого упало — падаем сразу, не продолжаем.

---

## обработка в handlers

```rust
// ✅ handler просто возвращает Result — AppError конвертируется в Response автоматически
pub async fn get_want_handler(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> AppResult<Json<WantResponse>> {
    let want = get_want(&pool, WantId(id)).await?;
    Ok(Json(WantResponse::from(want)))
}
```

---

## обработка в консьюмере брокера

```rust
async fn handle_message(payload: &[u8]) -> AppResult<()> {
    let envelope: MessageEnvelope = serde_json::from_slice(payload)
        .map_err(|e| AppError::InvalidPayload(e.to_string()))?;

    match envelope.event.as_str() {
        "parser.kwork.want" => handle_kwork_want(envelope.payload).await?,
        unknown => {
            warn!(event = %unknown, "unknown event, skipping");
        }
    }

    Ok(())
}

// в цикле консьюмера — один упавший message не роняет весь консьюмер
loop {
    let delivery = consumer.next().await?;
    if let Err(e) = handle_message(&delivery.data).await {
        error!(err = %e, "message handling failed");
        delivery.nack(BasicNackOptions::default()).await?;
    } else {
        delivery.ack(BasicAckOptions::default()).await?;
    }
}
```

---

## что запрещено

```rust
// ❌ unwrap в production коде
let val = option.unwrap();
let val = result.unwrap();

// ❌ expect без диагностики
let val = result.expect("oops");

// ✅ expect только при старте с понятным сообщением
let cfg = Config::from_env().expect("LOGGING__LEVEL must be set");

// ❌ паника
panic!("something went wrong");

// ❌ игнорируем ошибку
let _ = publish(payload).await;

// ✅ явно обрабатываем
if let Err(e) = publish(payload).await {
    warn!(err = %e, "publish failed, will retry");
}
```