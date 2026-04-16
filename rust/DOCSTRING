# docstrings conventions — rust

соглашения по документированию кода для rust-сервисов leadar.
используем `///` doc comments. язык — английский.

---

## когда писать

| что | doc comment |
|---|---|
| `pub fn` / `pub async fn` | ✅ обязательно |
| `pub struct` | ✅ обязательно |
| `pub enum` и его варианты | ✅ обязательно |
| `pub trait` | ✅ обязательно |
| приватная `fn` | 🟡 если логика неочевидна |
| поля `pub struct` | 🟡 если неочевидно |
| тесты | ❌ не нужен |

---

## формат

первая строка — краткое описание (одна фраза, без точки в конце если одна строка).
пустая строка — затем детали, примеры, паники.

```rust
/// Fetch a single want by ID from the database.
///
/// Returns `None` if the want does not exist.
///
/// # Errors
///
/// Returns [`AppError::Database`] on query failure.
pub async fn get_want(pool: &PgPool, id: WantId) -> AppResult<Option<Want>> {
    ...
}
```

---

## секции

### `# Errors` — обязательно для Result

```rust
/// Parse window.stateData from SSR HTML page.
///
/// # Errors
///
/// Returns [`ParseStateDataError::MarkerNotFound`] if the stateData
/// marker is absent from the HTML.
/// Returns [`ParseStateDataError::InvalidJson`] if the extracted JSON
/// is malformed.
pub fn parse_state_data(html: &str) -> Result<StateData, ParseStateDataError> {
```

### `# Panics` — если функция может паниковать

```rust
/// Initialize tracing subscriber.
///
/// # Panics
///
/// Panics if called more than once.
pub fn init_tracing() {
```

### `# Examples` — для публичного API крейта

```rust
/// Format price with Russian locale.
///
/// # Examples
///
/// ```
/// assert_eq!(format_price(3000.0), "3 000 ₽");
/// ```
pub fn format_price(price: f64) -> String {
```

---

## структуры и enum

```rust
/// Parsed want data from a freelance marketplace.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Want {
    pub id: WantId,
    /// Display name of the project.
    pub name: String,
    /// Client's desired budget in RUB.
    pub price_limit: f64,
    pub status: WantStatus,
    pub source: Source,
}

/// Current status of a want on the marketplace.
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum WantStatus {
    /// Accepting freelancer offers.
    Active,
    /// No longer accepting offers.
    Archive,
    /// Project is closed.
    Closed,
}
```

---

## что не писать

```rust
// ❌ пересказ сигнатуры
/// Returns the id of the want.
pub fn id(&self) -> WantId { self.id }

// ❌ очевидные вещи
/// The name field.
pub name: String,

// ✅ объясняем неочевидное
/// Maximum allowed price set by the platform (3x of price_limit).
pub possible_price_limit: f64,
```

---

## `//` комментарии — только TODO

обычные `//` комментарии в коде не используем.
исключение — только TODO по формату из `CODESTYLE_RUST.md`:

```rust
// TODO(qu1nqqy): add retry with backoff — rate limiting not yet observed
```