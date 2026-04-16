# typing conventions — rust

соглашения по типизации для rust-сервисов leadar (backend).

---

## структуры — serde

для всех данных из RabbitMQ, REST, БД — явные типы, никогда `serde_json::Value`:

```rust
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

// ❌
fn handle(payload: serde_json::Value) { ... }

// ✅
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KworkWantPayload {
    pub source: String,
    pub want_id: i64,
    pub name: String,
    pub description: String,
    pub price_limit: f64,
    pub possible_price_limit: f64,
    pub category_id: i32,
    pub max_days: i32,
    pub status: WantStatus,
    pub kwork_count: i32,
    pub views: i32,
    pub hired_percent: f64,
    pub url: String,
    pub date_create: DateTime<Utc>,
    pub date_expire: DateTime<Utc>,
}
```

### derive — стандартный набор

```rust
// для доменных моделей
#[derive(Debug, Clone, PartialEq)]

// для API/broker моделей
#[derive(Debug, Clone, Serialize, Deserialize)]

// для DB моделей (sqlx)
#[derive(Debug, Clone, sqlx::FromRow)]
```

не добавляем derive вслепую — только то что реально используется.

---

## enum вместо строк

```rust
// ❌
pub struct Want {
    pub status: String,
    pub source: String,
}

// ✅
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, sqlx::Type)]
#[serde(rename_all = "lowercase")]
#[sqlx(type_name = "want_status", rename_all = "lowercase")]
pub enum WantStatus {
    Active,
    Archive,
    Closed,
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Source {
    Kwork,
    Fl,
    Upwork,
}
```

---

## newtype для ID

предотвращает перепутывание `WantId` и `CategoryId`:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, sqlx::Type)]
#[sqlx(transparent)]
pub struct WantId(pub i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, sqlx::Type)]
#[sqlx(transparent)]
pub struct CategoryId(pub i32);

// использование
async fn get_want(id: WantId) -> Result<Want, AppError> { ... }

// ❌ не перепутаешь
get_want(CategoryId(38));  // compile error
```

---

## Option и Result

```rust
// Option — когда значение может отсутствовать
pub struct Want {
    pub description: Option<String>,
    pub date_expire: Option<DateTime<Utc>>,
}

// Result — когда операция может упасть
async fn fetch_want(id: WantId) -> Result<Want, AppError> { ... }

// ❌ unwrap/expect в production коде
let want = result.unwrap();

// ✅ ? оператор
let want = fetch_want(id).await?;

// ✅ явная обработка
let want = fetch_want(id).await.map_err(|e| {
    AppError::Database(e)
})?;
```

---

## type aliases

для сложных типов — в `src/core/types.rs`:

```rust
use std::collections::HashMap;

pub type WantMap = HashMap<WantId, Want>;
pub type CategoryMap = HashMap<CategoryId, String>;
pub type AppResult<T> = Result<T, AppError>;
```

---

## модели — разделение по слоям

не используем одну структуру везде — у каждого слоя своя:

```rust
// src/models/want.rs

/// Доменная модель — то что храним в памяти и передаём между слоями
#[derive(Debug, Clone)]
pub struct Want {
    pub id: WantId,
    pub name: String,
    pub price_limit: f64,
    pub status: WantStatus,
    pub source: Source,
    pub date_create: DateTime<Utc>,
}

/// DB модель — то что читаем из postgres через sqlx
#[derive(Debug, sqlx::FromRow)]
pub struct WantRow {
    pub id: i64,
    pub name: String,
    pub price_limit: f64,
    pub status: String,
    pub source: String,
    pub date_create: DateTime<Utc>,
}

/// REST ответ — то что отдаём клиенту
#[derive(Debug, Serialize)]
pub struct WantResponse {
    pub id: i64,
    pub name: String,
    pub price_limit: f64,
    pub status: String,
    pub source: String,
    pub date_create: String,
}

impl From<Want> for WantResponse { ... }
impl From<WantRow> for Want { ... }
```

---

## generics — только когда реально нужны

```rust
// ❌ generics ради generics
fn process<T: Into<String>>(val: T) -> String {
    val.into()
}

// ✅ конкретный тип
fn process(val: impl Into<String>) -> String {
    val.into()
}

// ✅ generics оправданы — переиспользуемый репозиторий
pub struct Repository<T> {
    pool: PgPool,
    _marker: PhantomData<T>,
}
```