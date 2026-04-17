# auth

соглашения по авторизации в leadar.
проект приватный — доступ по whitelist `telegram_id`.

---

## стратегия

| сервис | механизм |
|---|---|
| `backend` | выдаёт JWT после верификации Telegram Login |
| `frontend` | Telegram Login Widget → JWT в httpOnly cookie |
| `telegram-bot` | middleware по `allowed_ids` из env |

---

## telegram login widget (frontend)

пользователь нажимает "войти через Telegram" на сайте.
Telegram отдаёт callback object:

```ts
interface TelegramAuthData {
    id: number
    first_name: string
    username?: string
    photo_url?: string
    auth_date: number
    hash: string
}
```

frontend отправляет этот объект на `POST /auth/telegram`.

---

## верификация на backend (rust)

backend проверяет подлинность данных через HMAC-SHA256:

```rust
// алгоритм верификации Telegram Login Widget
// https://core.telegram.org/widgets/login#checking-authorization

fn verify_telegram_auth(data: &TelegramAuthData, bot_token: &str) -> bool {
    // 1. собираем строку из полей (кроме hash), отсортированных по ключу
    let check_string = build_check_string(data);  // "auth_date=...\nfirst_name=...\nid=..."

    // 2. secret_key = SHA256(bot_token)
    let secret_key = sha256(bot_token);

    // 3. HMAC-SHA256(check_string, secret_key) == data.hash
    hmac_sha256(&check_string, &secret_key) == data.hash
}
```

дополнительно проверяем:
- `auth_date` не старше 24 часов (защита от replay атак)
- `telegram_id` есть в `ALLOWED_TELEGRAM_IDS`

---

## jwt

после успешной верификации backend выдаёт JWT:

```rust
// payload
{
    "sub": "123456789",   // telegram_id как строка
    "iat": 1234567890,    // issued at
    "exp": 1234567890,    // expiry (iat + JWT_EXPIRY_HOURS * 3600)
}
```

### передача токена

JWT устанавливается как httpOnly cookie:

```rust
Cookie::build("token", jwt)
    .http_only(true)
    .secure(true)       // только HTTPS в проде
    .same_site(SameSite::Strict)
    .max_age(Duration::hours(jwt_expiry_hours))
    .finish()
```

**JWT не передаётся в теле ответа и не в Authorization header** — только httpOnly cookie.

### валидация на каждый запрос

axum middleware читает cookie `token`, верифицирует подпись, кладёт `telegram_id` в extension:

```rust
// в handlers
pub async fn get_wants_handler(
    Extension(user): Extension<AuthUser>,
    // ...
) -> AppResult<Json<WantsResponse>> { ... }

pub struct AuthUser {
    pub telegram_id: i64,
}
```

---

## telegram bot

middleware в aiogram — перед любым handler:

```python
ALLOWED_IDS: set[int] = {
    int(id_) for id_ in settings.auth.allowed_telegram_ids.split(",")
}

async def auth_middleware(handler, event, data):
    user = data.get("event_from_user")
    if user is None or user.id not in ALLOWED_IDS:
        # отвечаем шаблоном и не вызываем handler
        await event.answer(
            "Leadar пока что работает в закрытом режиме. "
            "Доступ по подписке — скоро."
        )
        return
    return await handler(event, data)
```

---

## эндпоинты auth

| метод | путь | описание |
|---|---|---|
| `POST` | `/auth/telegram` | верификация TG login data, выдача JWT cookie |
| `POST` | `/auth/logout` | удаление cookie (Set-Cookie с истёкшим max-age) |

`/auth/telegram` — единственный публичный эндпоинт кроме `/health` и `/metrics`.

---

## будущее — монетизация

при введении платного доступа:
- `users.is_active` в `leadar_backend` становится флагом активной подписки
- middleware backend проверяет `is_active` после JWT валидации
- `ALLOWED_TELEGRAM_IDS` остаётся для admin-доступа
- отдельный платёжный микросервис управляет `is_active`
