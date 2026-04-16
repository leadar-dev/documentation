# logging conventions — rust

соглашения по логгированию для rust-сервисов leadar.
используем `tracing` + `tracing-subscriber` с JSON форматом.

---

## инициализация

```rust
// src/main.rs
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

fn init_tracing() {
    tracing_subscriber::registry()
        .with(EnvFilter::from_env("LOGGING__LEVEL"))
        .with(fmt::layer().json())
        .init();
}
```

---

## уровни

| уровень | когда |
|---|---|
| `trace!` | очень детально — обычно отключён в проде |
| `debug!` | аргументы функций, промежуточные шаги |
| `info!` | успешное завершение операции, старт/стоп |
| `warn!` | нежелательная но обрабатываемая ситуация |
| `error!` | ошибка которую поймали и обработали |

---

## #[instrument] — основной инструмент

автоматически создаёт span и биндит аргументы:

```rust
use tracing::instrument;

// биндим только нужные поля, пропускаем тяжёлые (db pool, client)
#[instrument(skip(pool), fields(want_id = %id))]
pub async fn get_want(pool: &PgPool, id: WantId) -> AppResult<Want> {
    debug!("fetching want");

    let want = query_want(pool, id).await?;

    info!(name = %want.name, price = want.price_limit, "want fetched");
    Ok(want)
}
```

### skip — что пропускать

```rust
// ❌ логируем весь pool — бессмысленно и шумно
#[instrument]
async fn save(pool: &PgPool, want: &Want) { ... }

// ✅ пропускаем инфраструктуру, биндим только полезное
#[instrument(skip(pool, self), fields(want_id = %want.id))]
async fn save(&self, pool: &PgPool, want: &Want) { ... }
```

---

## поля в логах

```rust
// строковые значения — %val (Display)
info!(name = %want.name, status = %want.status, "want saved");

// числа и bool — val без префикса
debug!(price = want.price_limit, count = items.len(), "processed");

// debug-форматирование — ?val
debug!(want = ?want, "raw struct");  // только для debug уровня

// err — специальное поле для ошибок
error!(err = %e, want_id = %id, "failed to save want");
```

---

## span вручную — для циклов и сложных операций

```rust
use tracing::{info_span, Instrument};

pub async fn fetch_listing(category_id: CategoryId, pages: u32) -> AppResult<Vec<Want>> {
    let span = info_span!("fetch_listing", category_id = %category_id, pages);

    async move {
        info!("start");
        let mut results = vec![];

        for page in 1..=pages {
            let page_span = debug_span!("fetch_page", page);

            async {
                debug!("fetching");
                let wants = fetch_page(category_id, page).await?;
                debug!(count = wants.len(), "done");
                results.extend(wants);
                Ok::<_, AppError>(())
            }
            .instrument(page_span)
            .await?;
        }

        info!(total = results.len(), "listing done");
        Ok(results)
    }
    .instrument(span)
    .await
}
```

---

## логирование ошибок

логируем **один раз** — там где обрабатываем:

```rust
// ❌ логируем на каждом уровне
fn parse(html: &str) -> AppResult<StateData> {
    find_marker(html).map_err(|e| {
        error!(err = %e, "marker not found");  // лог 1
        e
    })
}

async fn run() {
    match parse(html) {
        Err(e) => error!(err = %e, "parse failed"),  // лог 2 — дубль
        Ok(d) => process(d),
    }
}

// ✅ логируем там где принимаем решение
fn parse(html: &str) -> AppResult<StateData> {
    find_marker(html)  // просто пробрасываем
}

async fn run() {
    match parse(html) {
        Err(e) => {
            error!(err = %e, "parse failed");
            // здесь решаем — retry, skip, exit
        }
        Ok(d) => process(d),
    }
}
```

---

## стандартные события

```rust
// старт сервиса
info!(host = %cfg.http.host, port = cfg.http.port, "server started");

// подключение к БД
info!(url = %cfg.database.url, "database connected");

// получение сообщения из брокера
debug!(routing_key = %key, size = payload.len(), "message received");

// публикация в брокер
debug!(routing_key = %key, "message published");

// HTTP запрос (в middleware)
info!(method = %method, path = %path, status = status.as_u16(), latency_ms = latency, "request");
```