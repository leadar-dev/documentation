# code style conventions — svelte / typescript

соглашения по стилю кода для svelte frontend leadar.

---

## стек

- **Svelte 5** + **TypeScript** + **Vite**
- CSS — без препроцессора, нативный scoped CSS в `.svelte` файлах
- никаких UI библиотек — пишем компоненты сами

---

## нейминг

| что | стиль | пример |
|---|---|---|
| переменная, функция | `camelCase` | `wantId`, `fetchWants` |
| константа модуля | `UPPER_SNAKE` | `API_BASE_URL`, `MAX_RETRIES` |
| тип, интерфейс | `PascalCase` | `Want`, `WantsResponse` |
| svelte компонент (файл) | `PascalCase` | `WantCard.svelte`, `FilterPanel.svelte` |
| store | `camelCase` + суффикс `Store` | `wantsStore`, `filtersStore` |
| ts файл | `kebab-case` | `api-client.ts`, `format.ts` |
| css класс | `kebab-case` | `.want-card`, `.filter-panel__title` |

---

## структура проекта

```
frontend/
  index.html
  vite.config.ts
  tsconfig.json
  package.json
  src/
    types.ts                  — все интерфейсы и типы
    app.svelte                — корневой компонент
    main.ts                   — точка входа, mount app
    core/
      constants.ts            — SOURCE, WANT_STATUS, API_BASE_URL
      errors.ts               — ApiError, NetworkError
    api/
      wants.ts                — fetchWants, fetchWant
      analytics.ts            — fetchZScore, fetchHeatmap
    store/
      wants.ts                — wantsStore, totalPages
      filters.ts              — filtersStore
    components/
      WantCard.svelte
      FilterPanel.svelte
      Pagination.svelte
      ErrorMessage.svelte
      Spinner.svelte
    utils/
      format.ts               — formatPrice, formatDate, truncate
```

---

## структура svelte компонента

```svelte
<!-- строго этот порядок блоков -->

<script lang="ts">
    // 1. импорты
    import type { Want } from "../types"
    import { formatPrice, formatDate } from "../utils/format"
    import { wantsStore } from "../store/wants"

    // 2. props
    export let want: Want
    export let compact: boolean = false

    // 3. реактивные переменные
    let expanded = false

    // 4. реактивные выражения
    $: priceFormatted = formatPrice(want.priceLimit)
    $: isExpired = new Date(want.dateExpire) < new Date()

    // 5. функции
    function toggleExpanded() {
        expanded = !expanded
    }
</script>

<!-- разметка -->
<article class="want-card" class:want-card--compact={compact}>
    <h3 class="want-card__title">{want.name}</h3>
    <span class="want-card__price">{priceFormatted}</span>
    {#if expanded}
        <p class="want-card__desc">{want.description}</p>
    {/if}
    <button on:click={toggleExpanded}>
        {expanded ? "Скрыть" : "Подробнее"}
    </button>
</article>

<!-- scoped стили — всегда в конце -->
<style>
    .want-card {
        border: 1px solid var(--color-border);
        border-radius: 8px;
        padding: 16px;
    }

    .want-card--compact {
        padding: 8px;
    }
</style>
```

---

## комментарии — запрещены

только TSDoc для публичных функций и TODO:

```ts
// TODO(qu1nqqy): add virtual scroll — perf not yet an issue
```

---

## async в компонентах

```svelte
<script lang="ts">
    import { onMount } from "svelte"
    import { wantsStore } from "../store/wants"

    // ❌ async onMount не обрабатывает ошибки
    onMount(async () => {
        await wantsStore.load()
    })

    // ✅ явный try/catch
    onMount(() => {
        wantsStore.load().catch(e => console.error(e))
    })
</script>
```

---

## реактивность — правила

```svelte
<script lang="ts">
    // ❌ мутируем массив напрямую — Svelte не заметит
    wants.push(newWant)

    // ✅ присваиваем новый массив
    wants = [...wants, newWant]

    // ❌ $: с побочными эффектами запросов
    $: fetchWants(filters)  // будет вызываться при каждом рендере

    // ✅ явный вызов при изменении
    $: if (filters) loadWants(filters)
</script>
```

---

## что запрещено

```ts
// ❌ var
var count = 0

// ❌ == вместо ===
if (id == "3") { ... }

// ❌ console.log в финальном коде
console.log("debug", data)

// ✅ только warn/error в обработчиках
console.error("fetch failed:", e)

// ❌ глобальные переменные
window.currentPage = 1
```