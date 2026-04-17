# state management — svelte / typescript

соглашения по управлению состоянием для svelte frontend leadar.  
используем нативные svelte stores — без внешних библиотек.

---

## принципы

- **единый источник правды** — состояние в store, не в компонентах
- **store на фичу** — `wantsStore`, `filtersStore`, не один гигантский
- **логика в store** — fetch, обработка ошибок, трансформации — не в компонентах
- **компоненты — только отображение** — читают store, вызывают экшены

---

## структура store

```ts
// src/store/wants.ts
import { writable, derived, get } from "svelte/store"
import type { Want, Filters, Pagination } from "../types"
import { fetchWants } from "../api/wants"
import { get as getFilters } from "./filters"

// --- типы ---

interface WantsState {
    wants: Want[]
    pagination: Pagination | null
    loading: boolean
    error: string | null
}

// --- store ---

const initialState: WantsState = {
    wants: [],
    pagination: null,
    loading: false,
    error: null,
}

export const wantsStore = writable<WantsState>(initialState)

// --- derived ---

export const hasMore = derived(
    wantsStore,
    $s => $s.pagination !== null && $s.pagination.page < $s.pagination.totalPages
)

// --- actions (экспортируемые функции) ---

export async function loadWants(): Promise<void> {
    wantsStore.update(s => ({ ...s, loading: true, error: null }))
    try {
        const data = await fetchWants(getFilters(filtersStore))
        wantsStore.update(s => ({ ...s, wants: data.data, pagination: data.pagination }))
    } catch (e) {
        wantsStore.update(s => ({ ...s, error: resolveError(e) }))
        console.error(e)
    } finally {
        wantsStore.update(s => ({ ...s, loading: false }))
    }
}

export function resetWants(): void {
    wantsStore.set(initialState)
}
```

---

## filters store

```ts
// src/store/filters.ts
import { writable } from "svelte/store"
import type { Filters } from "../types"
import { loadWants } from "./wants"

const initialFilters: Filters = {
    source: null,
    categoryId: null,
    priceMin: null,
    priceMax: null,
    page: 1,
}

export const filtersStore = writable<Filters>(initialFilters)

export async function setFilters(patch: Partial<Filters>): Promise<void> {
    filtersStore.update(f => ({ ...f, ...patch, page: 1 }))
    await loadWants()
}

export async function setPage(page: number): Promise<void> {
    filtersStore.update(f => ({ ...f, page }))
    await loadWants()
}

export function resetFilters(): void {
    filtersStore.set(initialFilters)
}
```

---

## использование в компонентах

```svelte
<script lang="ts">
    import { wantsStore, loadWants } from "../store/wants"
    import { filtersStore, setFilters } from "../store/filters"
    import { onMount } from "svelte"

    // Svelte 5: $derived для реактивной деструктуризации store
    const { wants, loading, error } = $derived($wantsStore)
    const filters = $derived($filtersStore)

    onMount(() => {
        loadWants().catch(console.error)
    })

    async function handleSourceChange(source: string) {
        await setFilters({ source: source as Source })
    }
</script>
```

---

## derived — вычисляемые значения

```ts
import { derived } from "svelte/store"
import { wantsStore } from "./wants"
import { filtersStore } from "./filters"

// фильтруем на клиенте если нужно
export const activeWants = derived(
    wantsStore,
    $s => $s.wants.filter(w => w.status === "active")
)

// комбинируем несколько store
export const isFiltered = derived(
    filtersStore,
    $f => $f.source !== null || $f.categoryId !== null
)
```

---

## правила

```ts
// ❌ мутируем state напрямую
wantsStore.update(s => {
    s.wants.push(newWant)
    return s
})

// ✅ возвращаем новый объект
wantsStore.update(s => ({
    ...s,
    wants: [...s.wants, newWant],
}))

// ❌ логика fetch прямо в компоненте
onMount(async () => {
    const data = await fetchWants(filters)
    wants = data.wants
})

// ✅ всё через store actions
onMount(() => {
    loadWants().catch(console.error)
})

// ❌ несколько компонентов делают одинаковый fetch
// ✅ один store — один fetch, компоненты читают результат

// ❌ Svelte 4 реактивность в Svelte 5
$: ({ wants } = $wantsStore)

// ✅ Svelte 5
const { wants } = $derived($wantsStore)
```