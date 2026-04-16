# docstrings conventions — svelte / typescript

соглашения по документированию кода для svelte frontend leadar.  
используем TSDoc. язык — английский.

---

## когда писать

| что | TSDoc |
|---|---|
| экспортируемая функция / action | ✅ обязательно |
| экспортируемый тип / интерфейс | 🟡 если неочевидно |
| store action | ✅ обязательно |
| svelte компонент (props) | ✅ обязательно |
| приватная функция | 🟡 если логика неочевидна |
| однострочная очевидная функция | ❌ не нужен |

---

## формат — TSDoc

```ts
/**
 * Fetch paginated list of wants from backend API.
 *
 * @param filters - active filter state from filtersStore
 * @param signal - optional AbortController signal
 * @returns paginated wants response
 * @throws {ApiError} on non-2xx response
 * @throws {NetworkError} if server is unreachable
 */
export async function fetchWants(
    filters: Filters,
    signal?: AbortSignal,
): Promise<WantsResponse> { ... }
```

---

## store actions

```ts
/**
 * Load wants from API using current filter state.
 * Updates wantsStore with results, loading state, and errors.
 * Aborts any in-flight request before starting a new one.
 */
export async function loadWants(): Promise<void> { ... }

/**
 * Update filters and reload wants from page 1.
 * @param patch - partial filter update to apply
 */
export async function setFilters(patch: Partial<Filters>): Promise<void> { ... }
```

---

## svelte компоненты — props через комментарий

в Svelte нет встроенного механизма TSDoc для props,  
документируем строчным комментарием перед prop:

```svelte
<script lang="ts">
    import type { Want } from "../types"

    /** Want data to display. */
    export let want: Want

    /** Render in compact mode without description. */
    export let compact: boolean = false

    /** Callback when user clicks the card. */
    export let onClick: ((want: Want) => void) | undefined = undefined
</script>
```

---

## интерфейсы и типы

```ts
/** Paginated API response for want listings. */
export interface WantsResponse {
    data: Want[]
    pagination: Pagination
}

/**
 * Active filter state for want listings.
 * null values mean "no filter applied".
 */
export interface Filters {
    source: Source | null
    categoryId: number | null
    priceMin: number | null
    priceMax: number | null
    page: number
}
```

---

## что не писать

```ts
// ❌ пересказ имени
/**
 * Gets the wants.
 */
export function getWants() { ... }

// ❌ TSDoc без информации
/**
 * @param id id
 * @returns want
 */
async function fetchWant(id: number): Promise<Want> { ... }

// ✅ документируем только неочевидное
/**
 * Fetch single want by ID.
 * @throws {ApiError} with status 404 if want does not exist
 */
async function fetchWant(id: number): Promise<Want> { ... }
```

---

## `//` комментарии — только TODO

```ts
// TODO(qu1nqqy): add virtual scroll for large lists — perf not yet an issue
```