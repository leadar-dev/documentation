# code style conventions

соглашения по стилю кода для python-сервисов leadar.

---

## комментарии — запрещены

комментарии в коде не используем. вместо них:

- **говорящие имена** — переменных, функций, классов
- **docstring** — для публичного API
- **TODO** — для техдолга (строго по формату ниже)

```python
# ❌
# получаем данные из html
marker = "window.stateData="
idx = r.text.find(marker)  # ищем начало

# ✅ — код говорит сам за себя
STATE_DATA_MARKER = "window.stateData="
marker_pos = html.find(STATE_DATA_MARKER)
```

---

## TODO — формат техдолга

единственный разрешённый вид комментария. строго по шаблону:

```python
# TODO(qu1nqqy): <что сделать> — <почему сейчас не сделано>
```

### примеры

```python
# TODO(qu1nqqy): add exponential backoff on 429 — rate limiting not yet needed
# TODO(qu1nqqy): replace TypedDict with pydantic model — validation needed later
# TODO(qu1nqqy): cache stateData extraction — profiling first
```

### правила

- **автор** — никнейм кто оставил, чтобы знать к кому идти
- **что** — конкретное действие, не размытое "улучшить"
- **почему не сейчас** — обоснование откладывания
- один TODO — одна задача
- закрытый TODO — удаляем из кода, не оставляем висеть

### что не писать

```python
# ❌ без автора
# TODO: fix this

# ❌ без обоснования
# TODO(qu1nqqy): add retry

# ❌ размытая задача
# TODO(qu1nqqy): improve performance somehow
```

---

## именование

| что | стиль | пример |
|---|---|---|
| переменная | `snake_case` | `project_id`, `state_data` |
| константа | `UPPER_SNAKE` | `STATE_DATA_MARKER`, `BASE_URL` |
| функция | `snake_case` | `fetch_project`, `parse_wants` |
| класс | `PascalCase` | `KworkParser`, `ParseError` |
| приватное | `_snake_case` | `_extract_json`, `_client` |
| type alias | `PascalCase` | `WantId`, `RawStateData` |

---

## структура файла

порядок блоков в файле:

```python
from __future__ import annotations          # 1. future

import json                                  # 2. stdlib
import re
from datetime import datetime

import httpx                                 # 3. third-party
import structlog

from src.core.types import RawStateData      # 4. internal (абсолютные импорты)
from src.services.logger import get_logger

SOME_CONSTANT = "value"                      # 5. константы модуля

logger = get_logger().bind(module=__name__)  # 6. модульный логгер


class SomeClass:                             # 7. классы
    ...


def some_function() -> None:                 # 8. функции
    ...
```

---

## прочее

- длина строки — **100 символов** (настроено в ruff)
- кавычки — **двойные** (`"string"`)
- trailing comma — **везде** где возможно (ruff форматирует автоматически)
- `if __name__ == "__main__"` — только в точках входа, не в модулях