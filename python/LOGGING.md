# logging conventions

соглашения по логгированию для python-сервисов leadar.  
используем `structlog` + `StructLogger` с методом `.bind()` для контекста.

---

## уровни логов

| уровень | когда использовать |
|---|---|
| `debug` | входящие аргументы, промежуточные значения, внутренние шаги |
| `info` | успешное завершение операции, старт/стоп сервиса, ключевые события |
| `warning` | ожидаемые, но нежелательные ситуации (пустой ответ, retry, rate limit) |
| `error` | ошибка, которую можно обработать (http 4xx/5xx, failed parse) |
| `critical` | ошибка, которая роняет сервис (нет подключения к БД, нет брокера) |

---

## стандартные поля

каждый лог должен содержать набор полей в зависимости от контекста.

### всегда

```python
log = logger.bind(
    service="parser-kwork",   # имя сервиса (parser-kwork, telegram-bot, backend)
    module="kwork.parser",    # __name__ модуля
)
```

### для функций и методов

биндим при входе в функцию:

```python
log = logger.bind(
    func="fetch_project",
    # входящие аргументы которые полезны для отладки
    project_id=project_id,
    category_id=category_id,
)
log.debug("start")
```

при выходе — логируем результат:

```python
log.info("done", price=want["priceLimit"], name=want["name"])
```

### для http запросов

```python
log = logger.bind(
    func="fetch_listing",
    url=url,
    method="GET",
    page=page,
)
log.debug("request")

# после ответа
log.info("response", status=response.status_code, size=len(response.content))
```

### для ошибок

всегда передаём `exc_info=True` чтобы structlog поймал traceback:

```python
try:
    data = parse_state_data(html)
except Exception as e:
    log.error("parse failed", error=str(e), exc_info=True)
    raise
```

---

## паттерны использования

### базовый — модульный логгер

в каждом файле создаём логгер с биндом модуля:

```python
from src.services.logger import get_logger

logger = get_logger().bind(
    service="parser-kwork",
    module=__name__,
)
```

### функциональный — локальный bind

внутри функции создаём локальный логгер, не мутируем глобальный:

```python
async def fetch_project(project_id: int) -> dict:
    log = logger.bind(func="fetch_project", project_id=project_id)
    log.debug("start")

    response = await client.get(url)
    log.debug("got response", status=response.status_code)

    data = parse(response)
    log.info("done", name=data["name"], price=data["priceLimit"])

    return data
```

### для классов — bind в `__init__`

```python
class KworkParser:
    def __init__(self, category_id: int) -> None:
        self.log = logger.bind(
            component="KworkParser",
            category_id=category_id,
        )

    async def fetch_page(self, page: int) -> list[dict]:
        log = self.log.bind(func="fetch_page", page=page)
        log.debug("start")
        ...
        log.info("done", count=len(wants))
```

### для циклов и пагинации

```python
for page in range(1, total_pages + 1):
    log = logger.bind(func="fetch_listing", page=page, total=total_pages)
    log.debug("fetching page")
    ...
    log.info("page done", wants_count=len(wants))
```

---

## что биндить — шпаргалка

| ситуация | поля |
|---|---|
| вход в функцию | `func`, ключевые аргументы |
| http запрос | `url`, `method`, `page` (если есть) |
| http ответ | `status`, `size` |
| парсинг | `source`, кол-во найденных записей |
| успех | итоговые значения (`id`, `name`, `price`) |
| ошибка | `error=str(e)`, `exc_info=True` |
| retry | `attempt`, `max_attempts`, причина |
| публикация в rabbitmq | `queue`, `routing_key`, `message_id` |

---

## чего не делать

```python
# ❌ не логируем чувствительные данные
log.debug("user data", password=user.password, token=token)

# ❌ не используем f-string в event
log.info(f"fetched {count} wants")  # теряем структуру

# ✅ правильно
log.info("fetched wants", count=count)

# ❌ не мутируем глобальный логгер
logger = logger.bind(page=page)  # перетрёт контекст

# ✅ создаём локальный
log = logger.bind(page=page)
```

---

## пример полного лога парсера

```python
# parser-kwork/src/kwork/parser.py

from src.services.logger import get_logger

logger = get_logger().bind(service="parser-kwork", module=__name__)


class KworkParser:
    def __init__(self) -> None:
        self.log = logger.bind(component="KworkParser")

    async def fetch_listing(self, category_id: int, pages: int) -> list[dict]:
        log = self.log.bind(func="fetch_listing", category_id=category_id, pages=pages)
        log.info("start")

        results = []
        for page in range(1, pages + 1):
            page_log = log.bind(page=page)
            page_log.debug("fetching page")

            try:
                response = await self.client.get(url, params={"page": page})
                page_log.debug("got response", status=response.status_code)

                wants = self.parse_wants(response.text)
                page_log.info("page done", count=len(wants))
                results.extend(wants)

            except Exception as e:
                page_log.error("page failed", error=str(e), exc_info=True)
                continue

        log.info("listing done", total=len(results))
        return results
```

---

> про rust — отдельный `logging_rust.md` когда доберёмся до `tracing` крейта.
