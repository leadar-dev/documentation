# code style conventions — rust

соглашения по стилю кода для rust-сервисов leadar.

---

## нейминг

| что | стиль | пример |
|---|---|---|
| переменная, функция, метод | `snake_case` | `want_id`, `fetch_want`, `get_by_id` |
| тип, трейт, enum | `PascalCase` | `WantStatus`, `WantRepository` |
| вариант enum | `PascalCase` | `WantStatus::Active` |
| константа, static | `SCREAMING_SNAKE` | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT` |
| модуль, файл | `snake_case` | `want.rs`, `mod analytics` |
| lifetime | короткий | `'a`, `'db`, `'req` |
| generic параметр | `PascalCase` один символ или слово | `T`, `E`, `Item` |

---

## структура проекта

```
backend/
  Cargo.toml
  Cargo_template.toml
  migrations/
    0001_create_wants.sql
    0002_create_categories.sql
  src/
    main.rs           — точка входа, сборка app, init_tracing, serve
    lib.rs            — реэкспорты публичного API крейта
    errors.rs         — AppError + IntoResponse
    config.rs         — Config через envy
    router.rs         — axum Router, регистрация handlers
    db/
      mod.rs          — PgPool хелперы, run_migrations
      wants.rs        — SQL queries для wants
      categories.rs
    handlers/
      mod.rs
      wants.rs        — axum handlers
      analytics.rs
    services/
      mod.rs
      analytics.rs    — z-score, heatmap бизнес-логика
    broker/
      mod.rs
      consumer.rs     — RabbitMQ консьюмер
      publisher.rs
    models/
      mod.rs
      want.rs         — доменные структуры + WantRow + WantResponse
      category.rs
    core/
      types.rs        — type aliases, newtype IDs
```

### правила структуры

- `mod.rs` — только `pub use` реэкспорты, никакой логики
- один файл — одна зона ответственности
- `pub` только для того что реально нужно снаружи
- внутренние хелперы — `pub(crate)` или приватные

---

## комментарии

комментарии в коде не используем — код должен быть самодокументированным.  
исключения:

```rust
/// Doc comment для публичного API (fn, struct, enum, trait)
pub fn fetch_want(id: WantId) -> AppResult<Want> { ... }

// TODO(qu1nqqy): <что сделать> — <почему не сейчас>
// TODO(qu1nqqy): add retry with exponential backoff — rate limiting not yet observed
```

---

## clippy

в `Cargo.toml`:

```toml
[lints.clippy]
pedantic = "warn"
unwrap_used = "deny"
expect_used = "warn"
panic = "warn"
clone_on_ref_ptr = "warn"
```

запускаем перед каждым PR:

```bash
cargo clippy -- -D warnings
```

---

## форматирование

используем `rustfmt` с дефолтными настройками:

```bash
cargo fmt
```

`rustfmt.toml` не заводим — стандартный стиль.

---

## imports — порядок

`rustfmt` сортирует автоматически. группировка:

```rust
// 1. std
use std::collections::HashMap;
use std::sync::Arc;

// 2. внешние крейты
use axum::{Json, extract::{Path, State}};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

// 3. internal
use crate::errors::AppError;
use crate::models::want::{Want, WantRow, WantResponse};
use crate::core::types::{AppResult, WantId};
```

---

## async — правила

```rust
// ❌ блокирующий код в async контексте
async fn fetch(url: &str) -> AppResult<String> {
    let body = std::fs::read_to_string("cache.txt")?;  // блокирует tokio runtime
    ...
}

// ✅ tokio::fs для I/O в async
async fn fetch(url: &str) -> AppResult<String> {
    let body = tokio::fs::read_to_string("cache.txt").await?;
    ...
}

// ✅ тяжёлые CPU задачи — spawn_blocking
let result = tokio::task::spawn_blocking(|| {
    heavy_computation()
}).await?;
```

---

## clone — только по необходимости

```rust
// ❌ clone без причины
let name = want.name.clone();
log_name(&name);
process(&want);

// ✅ передаём ссылку
log_name(&want.name);
process(&want);

// ✅ clone оправдан — Arc для шаринга между задачами
let pool = Arc::new(pool);
let pool_clone = Arc::clone(&pool);
tokio::spawn(async move { use_pool(pool_clone).await });
```
