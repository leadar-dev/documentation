# gitflow conventions

соглашения по git для всех репозиториев leadar.

---

## ветки

```
main        — стабильный прод, только через PR из dev
dev         — интеграционная ветка, только через PR из feature/*
feature/*   — разработка, создаём от dev, мержим в dev
fix/*       — хотфикс бага в dev
refactor/*  — рефакторинг без новой функциональности
docs/*      — только документация
chore/*     — зависимости, конфиги, тулинг
```

### правила

- **direct push в `main` и `dev` — запрещён**
- ветка живёт пока не смержена — после merge удаляем
- одна ветка — одна логическая задача
- называем ветку по тому что делаем, не по файлам

---

## стратегии мержа

| PR | стратегия | почему |
|---|---|---|
| `feature/*` → `dev` | **squash merge** | чистая история dev, один коммит на задачу |
| `dev` → `main` | **merge commit** | сохраняет связь веток, следующий PR всегда без конфликтов |

**squash dev → main — запрещён.** Squash создаёт в main коммит без связи с историей dev. Общий предок веток сбрасывается к Initial commit, и каждый следующий спринт даёт add/add конфликты на всех изменённых файлах.

### GitHub branch protection

В настройках репо (`Settings → Branches → main`):
- `Allow merge commits` — ✅ включено
- `Allow squash merging` — ✅ включено (нужно для feature → dev)
- `Allow rebase merging` — по желанию

Стратегия dev → main контролируется договорённостью, не GitHub-ом: при открытии PR выбирать **"Create a merge commit"**.

```bash
# ✅
git checkout -b feature/kwork-listing-parser
git checkout -b fix/state-data-extraction
git checkout -b refactor/parser-base-class
git checkout -b docs/logging-conventions
git checkout -b chore/add-ruff-config

# ❌
git checkout -b fix123
git checkout -b ilya-branch
git checkout -b update
```

---

## спринты

спринт — это набор связанных фич/фиксов, которые вместе дают законченный результат.  
**завершение спринта = PR `dev → main` через merge commit**.

к датам не привязаны — спринт закрывается когда готов, не по расписанию.

### закрытие спринта — процедура

```bash
# 1. убедиться что dev актуален
git checkout dev
git pull origin dev

# 2. открыть PR dev → main на GitHub
# название: sprint-N: <одна фраза что готово>

# 3. при мерже выбрать "Create a merge commit" — не squash, не rebase
# после мержа: main и dev имеют общего предка → следующий PR чистый
```

### деление спринтов

дробим мельче чем кажется нужным. критерий — спринт можно описать одним предложением.

```
sprint-1: базовая инфраструктура
  infrastructure: docker compose, postgres, rabbitmq, nginx, dragonfly
  postgres: инициализация баз, пользователи
  rabbitmq: exchanges, queues, DLQ конфиг (definitions.json)
  nginx: сабдомены api.leadar / leadar, /metrics закрыт
  prometheus + postgres-exporter: базовый мониторинг
  healthcheck: для всех сервисов в compose

sprint-2: парсер kwork — листинг
  parser-kwork: SSR парсинг, извлечение wants, пагинация
  parser-kwork: дедупликация через dragonfly, публикация в rabbitmq

sprint-3: парсер kwork — детали заказа
  parser-kwork: парсинг одного заказа, нормализация полей, TypedDict модели

sprint-4: backend — приём событий и авторизация
  backend: консьюмер rabbitmq, upsert wants, публикация backend.want.new
  backend: REST endpoint /wants, /categories
  backend: авторизация — Telegram Login Widget, JWT httpOnly cookie
  backend: /health, /metrics (axum-prometheus)
  backend: DLQ consumer
  ci: GitHub Actions — сборка образов, push в GHCR, деплой на VPS

sprint-5: аналитика — z-score
  backend: batch-пересчёт z-score по расписанию (tokio interval)
  backend: endpoint /analytics/zscore, /analytics/heatmap

sprint-6: frontend — фид заказов
  frontend: Svelte 5, листинг wants, фильтры, пагинация
  frontend: авторизация через Telegram Login Widget

sprint-7: telegram bot — базовый
  telegram-bot: авторизация по allowed_ids, шаблон закрытого доступа
  telegram-bot: подписка на категории, уведомления через bot.notifications

sprint-8: heatmap и тренды
  backend + frontend: тепловая карта активности, линейные тренды

sprint-9: парсер fl.ru (опционально)
  parser-fl: аналогично kwork

sprint-10: парсер upwork (опционально)
  parser-upwork: требует отдельного ресёрча (anti-bot, API)
```

---

## жизненный цикл ветки

```bash
# 1. берём свежий dev
git checkout dev
git pull origin dev

# 2. создаём ветку
git checkout -b feature/kwork-listing-parser

# 3. коммитим по мере работы
git add .
git commit  # через conv commits плагин

# 4. пушим и открываем PR в dev
git push origin feature/kwork-listing-parser

# 5. после merge — удаляем ветку
git branch -d feature/kwork-listing-parser
```

---

## conventional commits

формат:

```
type(scope): глагол прошедшего времени + что изменилось
```

максимум 74 символа, на русском, без кавычек и markdown.

### типы

| тип | когда |
|---|---|
| `feat` | новая функциональность |
| `fix` | исправление бага |
| `refactor` | рефакторинг без изменения поведения |
| `docs` | только документация |
| `chore` | зависимости, конфиги, тулинг |
| `test` | тесты |
| `perf` | оптимизация производительности |
| `ci` | github actions, docker, деплой |

### скоупы

скоуп = название сервиса или слоя:

```
parser, listing, bot, backend, infra, auth, analytics, db, broker, logger, config
```

### примеры

```
feat(listing): добавил пагинацию по страницам категорий
feat(parser): реализовал извлечение stateData из SSR HTML
fix(parser): исправил маркер поиска window.stateData без пробелов
refactor(parser): вынес парсинг JSON в отдельный метод
chore(parser): добавил httpx и beautifulsoup4 в зависимости
docs(logger): добавил соглашения по логгированию
ci(infra): настроил docker compose для rabbitmq с vhost
perf(listing): добавил кэш stateData на время сессии
```

---

## PR правила

### feature/* → dev

- стратегия мержа: **squash merge**
- название PR = то же что коммит, но без ограничения в 74 символа
- описание — что сделано и почему, если неочевидно
- самомерж разрешён (работаем соло), но после паузы и review глазами

### dev → main (закрытие спринта)

- стратегия мержа: **merge commit** — выбирать явно в интерфейсе GitHub
- название PR: `sprint-N: <одна фраза что готово>`

примеры:

```
sprint-2: парсер kwork — листинг заказов
sprint-4: backend — приём и хранение заказов
```

описание PR спринта — список веток которые вошли:

```
## что вошло
- feature/kwork-state-data-extraction
- feature/kwork-listing-pagination
- feature/kwork-rabbitmq-publisher
- fix/state-data-marker-no-spaces
```

---

## .gitignore — базовый для python сервисов

```gitignore
.venv/
__pycache__/
*.pyc
*.pyo
.env
.env.*
!.env.example
dist/
*.egg-info/
.ruff_cache/
.mypy_cache/
.ty_cache/
```