# api contracts

соглашения по межсервисному взаимодействию в leadar.  
транспорт — RabbitMQ. формат — JSON.

---

## структура RabbitMQ

```
exchange: leadar.events   (topic, durable)

routing keys:
  parser.kwork.want       — новый/обновлённый заказ с kwork
  parser.fl.want          — новый/обновлённый заказ с fl.ru
  parser.upwork.want      — новый/обновлённый заказ с upwork

queues:
  backend.wants           — backend слушает все parser.*.want
  bot.notifications       — bot слушает backend.want.new
```

---

## envelope — обёртка для всех сообщений

каждое сообщение оборачиваем в единый конверт:

```json
{
  "event": "parser.kwork.want",
  "version": 1,
  "timestamp": "2026-04-11T01:42:16Z",
  "payload": { ... }
}
```

| поле | тип | описание |
|---|---|---|
| `event` | string | routing key события |
| `version` | int | версия схемы payload, начинаем с 1 |
| `timestamp` | ISO 8601 UTC | когда сформировано сообщение |
| `payload` | object | данные события |

---

## события

### `parser.kwork.want` — заказ с kwork

```json
{
  "event": "parser.kwork.want",
  "version": 1,
  "timestamp": "2026-04-11T01:42:16Z",
  "payload": {
    "source": "kwork",
    "want_id": 3148262,
    "name": "WordPress: настроить загрузку фото в записи, почистить",
    "description": "Оптимизировать систему загрузки изображений...",
    "price_limit": 3000.00,
    "possible_price_limit": 9000.00,
    "category_id": 38,
    "max_days": 10,
    "status": "active",
    "kwork_count": 7,
    "views": 173,
    "hired_percent": 43.0,
    "url": "https://kwork.ru/projects/3148262/view",
    "date_create": "2026-04-11T00:42:16Z",
    "date_expire": "2026-04-14T00:00:00Z"
  }
}
```

### `parser.fl.want` — заказ с fl.ru

схема аналогична `parser.kwork.want`, поле `source: "fl"`.  
специфичные для fl поля добавляем по мере изучения API.

### `parser.upwork.want` — заказ с upwork

схема аналогична, `source: "upwork"`.  
бюджет в USD — добавляем поле `currency: "USD"`.

---

## соглашения по полям

- **snake_case** для всех ключей
- **числа** — не строки: `price_limit: 3000.00`, не `"3000.00"`
- **даты** — ISO 8601 UTC: `"2026-04-11T00:42:16Z"`
- **nullable** — явный `null`, не отсутствие поля
- **source** — всегда указываем откуда пришёл заказ

---

## версионирование

при изменении схемы payload:

- **обратно совместимые изменения** (добавили поле) — `version` не меняем
- **breaking changes** (удалили/переименовали поле) — инкрементируем `version`
- старую версию поддерживаем пока все консьюмеры не обновлены

---

## python — TypedDict для envelope

```python
from __future__ import annotations

from typing import TypedDict


class MessageEnvelope(TypedDict):
    event: str
    version: int
    timestamp: str
    payload: dict[str, object]


class KworkWantPayload(TypedDict):
    source: str
    want_id: int
    name: str
    description: str
    price_limit: float
    possible_price_limit: float
    category_id: int
    max_days: int
    status: str
    kwork_count: int
    views: int
    hired_percent: float
    url: str
    date_create: str
    date_expire: str
```

---

## REST API (backend)

базовый URL: `http://backend:8000/api/v1`

### формат ответа

успех:

```json
{
  "ok": true,
  "data": { ... }
}
```

ошибка:

```json
{
  "ok": false,
  "error": {
    "code": "WANT_NOT_FOUND",
    "message": "want 3148262 not found"
  }
}
```

### эндпоинты (планируемые)

| метод | путь | описание |
|---|---|---|
| `GET` | `/wants` | список заказов с фильтрами |
| `GET` | `/wants/{id}` | один заказ |
| `GET` | `/analytics/zscore` | z-score по категориям |
| `GET` | `/analytics/heatmap` | тепловая карта активности |
| `GET` | `/categories` | список категорий |

### фильтры для `GET /wants`

```
?source=kwork
?category_id=38
?price_min=1000&price_max=10000
?status=active
?page=1&limit=20
```