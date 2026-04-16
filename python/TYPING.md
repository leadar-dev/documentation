# typing conventions

соглашения по типизации для python-сервисов leadar.  
используем `ty` как type checker, `from __future__ import annotations` везде.

---

## базовые правила

- `from __future__ import annotations` — первой строкой в каждом файле
- типизируем всё: аргументы, возвращаемые значения, поля классов
- исключение — тесты (`tests/` не проверяются через ty)
- `Any` — только с явным комментарием почему

```python
from __future__ import annotations

# ❌
def fetch(id):
    ...

# ✅
def fetch(id: int) -> dict[str, Any]:
    ...
```

---

## встроенные типы

используем встроенные дженерики (python 3.10+), не `typing`:

```python
# ❌
from typing import Dict, List, Optional, Tuple
def foo(items: List[str]) -> Optional[Dict[str, int]]: ...

# ✅
def foo(items: list[str]) -> dict[str, int] | None: ...
```

---

## None и Optional

```python
# ❌
from typing import Optional
def get(id: int) -> Optional[str]: ...

# ✅
def get(id: int) -> str | None: ...
```

---

## TypedDict для внешних данных

для данных из API/HTML (kwork stateData, rabbitmq payload и т.д.) — `TypedDict`:

```python
from __future__ import annotations

from typing import TypedDict


class WantUser(TypedDict):
    USERID: int
    username: str
    wants_hired_percent: str


class Want(TypedDict):
    id: int
    name: str
    description: str
    priceLimit: str
    possiblePriceLimit: int
    category_id: str
    max_days: str
    status: str
    kwork_count: int
    date_create: str
    date_expire: str
    views_dirty: str
    timeLeft: str
    user: WantUser
```

---

## dataclass для внутренних моделей

для моделей которые мы создаём сами — `dataclass` или `pydantic`:

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime


@dataclass
class ParsedWant:
    id: int
    name: str
    price_limit: float
    possible_price_limit: float
    category_id: int
    max_days: int
    status: str
    kwork_count: int
    date_create: datetime
    date_expire: datetime
    views: int
    hired_percent: float
```

---

## Any — только с обоснованием

```python
# ❌
def process(data: Any) -> Any:
    ...

# ✅ — с комментарием
def configure_logger() -> None:
    structlog.configure(
        processors=cast(Any, [...]),  # structlog не типизирует processors
    )
```

---

## callable и генераторы

```python
from collections.abc import AsyncGenerator, Callable

# callback
handler: Callable[[int, str], bool]

# async generator
async def paginate(url: str) -> AsyncGenerator[list[Want], None]:
    ...
    yield wants
```

---

## type aliases

для сложных типов создаём алиасы в `src/core/types.py`:

```python
from __future__ import annotations

from typing import TypeAlias

WantId: TypeAlias = int
CategoryId: TypeAlias = int
RawStateData: TypeAlias = dict[str, Any]
```

---

## cast

используем только когда ty не может вывести тип сам:

```python
from typing import cast

# stateData из HTML — мы знаем структуру, ty не знает
state = cast(RawStateData, json.loads(raw))
```