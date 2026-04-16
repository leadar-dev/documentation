# docstring conventions

соглашения по документированию кода для python-сервисов leadar.  
стиль — Google. язык — английский.

---

## когда писать docstring

| что | docstring |
|---|---|
| публичный класс | ✅ обязательно |
| публичный метод / функция | ✅ обязательно |
| приватный метод (`_foo`) | 🟡 если логика неочевидна |
| `__init__` | ❌ документируем в классе |
| property | 🟡 если неочевидно |
| тесты | ❌ не нужен |

---

## формат — Google style

```python
def fetch_project(project_id: int, retries: int = 3) -> Want:
    """Fetch a single project from kwork by ID.

    Parses SSR HTML page and extracts want data from window.stateData.

    Args:
        project_id: Kwork project identifier.
        retries: Number of retry attempts on network failure.

    Returns:
        Parsed want data as TypedDict.

    Raises:
        ParseError: If stateData is missing or malformed.
        httpx.HTTPStatusError: On non-2xx response.
    """
```

---

## класс

docstring на классе — описание + Attributes если есть нетривиальные поля:

```python
class KworkParser:
    """Parser for kwork.ru freelance marketplace.

    Fetches project listings and individual project pages,
    extracts structured data from SSR HTML.

    Attributes:
        client: Shared httpx async client.
        base_url: Kwork root URL.
    """

    def __init__(self, client: httpx.AsyncClient) -> None:
        self.client = client
        self.base_url = "https://kwork.ru"
```

---

## короткий однострочный docstring

если функция простая — одна строка, без Args/Returns:

```python
def is_want_active(want: Want) -> bool:
    """Return True if the project is currently accepting offers."""
    return want["status"] == "active"
```

---

## что не писать

```python
# ❌ очевидные вещи
def get_id(want: Want) -> int:
    """Returns the id of the want."""  # бесполезно — и так понятно из типов
    return want["id"]

# ❌ пересказ кода
def fetch(url: str) -> str:
    """Makes a GET request to url and returns response text."""

# ✅ объясняем ПОЧЕМУ, а не ЧТО
def fetch(url: str) -> str:
    """Fetch page content with browser-like headers to avoid bot detection."""
```

---

## raises — обязательно для публичных методов

если функция может бросить исключение — документируем:

```python
def parse_state_data(html: str) -> RawStateData:
    """Extract window.stateData from SSR HTML.

    Args:
        html: Raw HTML page content.

    Returns:
        Parsed stateData dictionary.

    Raises:
        ParseError: If stateData marker is not found in HTML.
        json.JSONDecodeError: If stateData contains invalid JSON.
    """
```

---

## async функции

никаких отличий от sync — просто документируем как обычно:

```python
async def fetch_listing(self, category_id: int, page: int) -> list[Want]:
    """Fetch one page of project listings for a given category.

    Args:
        category_id: Kwork category identifier.
        page: Page number, 1-indexed.

    Returns:
        List of want dicts parsed from stateData.
    """
```