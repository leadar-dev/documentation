# error handling — svelte / typescript

специфика обработки ошибок для svelte frontend. общие принципы — в `../ERRORS.md`.

---

## классы ошибок

```ts
// src/core/errors.ts

export class ApiError extends Error {
    constructor(
        public readonly status: number,
        public readonly code: string,
        message: string,
    ) {
        super(message)
        this.name = "ApiError"
    }
}

export class NetworkError extends Error {
    constructor() {
        super("Network unavailable")
        this.name = "NetworkError"
    }
}
```

---

## api слой — бросаем типизированные ошибки

```ts
export async function fetchWants(filters: Filters): Promise<WantsResponse> {
    let response: Response
    try {
        response = await fetch(`/wants?${buildParams(filters)}`)
    } catch {
        throw new NetworkError()
    }

    if (!response.ok) {
        const body = await response.json().catch(() => ({}))
        throw new ApiError(
            response.status,
            body.error?.code ?? "UNKNOWN",
            body.error?.message ?? "Request failed",
        )
    }

    return response.json()
}
```

---

## store — ошибки как состояние

ошибки храним в store, не пробрасываем в компоненты:

```ts
// src/store/wants.ts

export const wantsStore = writable<WantsState>(initialState)

export async function loadWants(): Promise<void> {
    wantsStore.update(s => ({ ...s, loading: true, error: null }))

    try {
        const data = await fetchWants(get(filtersStore))
        wantsStore.update(s => ({
            ...s,
            wants: data.data,
            pagination: data.pagination,
        }))
    } catch (e) {
        const message = e instanceof NetworkError
            ? "Нет соединения с сервером"
            : e instanceof ApiError && e.status === 404
                ? "Заказы не найдены"
                : "Ошибка загрузки заказов"

        wantsStore.update(s => ({ ...s, error: message }))
        console.error("[loadWants]", e)
    } finally {
        wantsStore.update(s => ({ ...s, loading: false }))
    }
}
```

---

## компонент — читает ошибку из store

```svelte
<!-- WantList.svelte -->
<script lang="ts">
    import { wantsStore, loadWants } from "../store/wants"
    import WantCard from "./WantCard.svelte"
    import ErrorMessage from "./ErrorMessage.svelte"
    import Spinner from "./Spinner.svelte"

    // Svelte 5: деструктурируем через $derived
    const { wants, loading, error } = $derived($wantsStore)
</script>

{#if loading}
    <Spinner />
{:else if error}
    <!-- Svelte 5: onretry вместо on:retry -->
    <ErrorMessage message={error} onretry={loadWants} />
{:else if wants.length === 0}
    <p class="empty">Заказов не найдено</p>
{:else}
    {#each wants as want (want.id)}
        <WantCard {want} />
    {/each}
{/if}
```

---

## AbortController — отмена запросов

```ts
// src/store/wants.ts
let controller: AbortController | null = null

export async function loadWants(): Promise<void> {
    controller?.abort()
    controller = new AbortController()

    wantsStore.update(s => ({ ...s, loading: true, error: null }))

    try {
        const data = await fetchWants(get(filtersStore), controller.signal)
        wantsStore.update(s => ({ ...s, wants: data.data, pagination: data.pagination }))
    } catch (e) {
        if (e instanceof Error && e.name === "AbortError") return
        wantsStore.update(s => ({ ...s, error: "Ошибка загрузки" }))
        console.error(e)
    } finally {
        wantsStore.update(s => ({ ...s, loading: false }))
    }
}
```

---

## глобальный обработчик

```ts
// src/main.ts
window.addEventListener("unhandledrejection", e => {
    console.error("[unhandledrejection]", e.reason)
    e.preventDefault()
})
```

---

## что запрещено

```ts
// ❌ пробрасываем ошибку в компонент через prop (Svelte 4)
export let error: Error | null

// ✅ ошибка как строка в store
error: string | null

// ❌ показываем техническую ошибку юзеру
errorMessage = e.message  // "Failed to fetch"

// ✅ понятный текст
errorMessage = "Нет соединения с сервером"
```