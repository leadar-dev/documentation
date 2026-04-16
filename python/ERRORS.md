# error handling — python

python-специфика обработки ошибок. общие принципы — в `../ERRORS.md`.

---

## иерархия исключений

базовое исключение на сервис, от него наследуем все остальные:

```python
from __future__ import annotations


class LeadarError(Exception):
    """Base exception for all leadar services."""


class ParserError(LeadarError):
    """Base parser exception."""

class ParseStateDataError(ParserError):
    """Failed to extract or parse window.stateData from HTML."""

class WantNotFoundError(ParserError):
    """Target want data key missing in stateData."""

class PageFetchError(ParserError):
    """HTTP request to marketplace failed."""


class BrokerError(LeadarError):
    """Base broker exception."""

class PublishError(BrokerError):
    """Failed to publish message to RabbitMQ."""
```

---

## raise X from e — всегда сохраняем цепочку

```python
# ❌ теряем оригинальный traceback
except httpx.HTTPStatusError:
    raise PageFetchError("request failed")

# ✅ сохраняем причину
except httpx.HTTPStatusError as e:
    raise PageFetchError(f"HTTP {e.response.status_code} for {url}") from e
```

---

## HTTP ошибки — httpx

```python
async def fetch_page(self, url: str) -> str:
    try:
        response = await self.client.get(url)
        response.raise_for_status()
        return response.text
    except httpx.HTTPStatusError as e:
        raise PageFetchError(
            f"HTTP {e.response.status_code} for {url}"
        ) from e
    except httpx.RequestError as e:
        raise PageFetchError(f"request failed for {url}") from e
```

---

## паттерн — skip одного элемента

```python
for project_id in project_ids:
    try:
        want = await fetch_project(project_id)
        results.append(want)
    except PageFetchError as e:
        log.warning("skipping project", project_id=project_id, error=str(e))
        continue
    except ParseStateDataError as e:
        log.warning("parse failed, skipping", project_id=project_id, error=str(e))
        continue
```

---

## паттерн — fail fast при старте

```python
async def on_startup() -> None:
    try:
        await db.connect()
    except Exception as e:
        log.critical("db connection failed", error=str(e), exc_info=True)
        raise SystemExit(1) from e
```

---

## что запрещено

```python
# ❌ голый except без обработки
try:
    data = parse(html)
except:
    pass

# ❌ Exception без контекста
raise Exception("something went wrong")

# ❌ логируем и глотаем
except Exception as e:
    log.error("error", error=str(e))
    # ничего не делаем дальше
```