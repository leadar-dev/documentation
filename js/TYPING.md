# typing conventions — svelte / typescript

соглашения по типизации для svelte frontend leadar.  
используем TypeScript — строгий режим, никаких `any`.

---

## базовые правила

- `strict: true` в `tsconfig.json` — всегда
- `any` — запрещён, используем `unknown` если тип неизвестен
- типы для всех публичных функций, props, store значений
- `interface` для объектов с методами или наследованием, `type` для остального

---

## types.ts — общие типы

все переиспользуемые типы в одном месте:

```ts
// src/types.ts

export interface Want {
    id: number
    name: string
    description: string
    priceLimit: number
    possiblePriceLimit: number
    status: WantStatus
    source: Source
    categoryId: number
    maxDays: number
    kworkCount: number
    views: number
    hiredPercent: number
    url: string
    dateCreate: string
    dateExpire: string
}

export interface Category {
    id: number
    name: string
}

export interface Filters {
    source: Source | null
    categoryId: number | null
    priceMin: number | null
    priceMax: number | null
    page: number
}

export interface Pagination {
    page: number
    limit: number
    total: number
    totalPages: number
}

export interface WantsResponse {
    data: Want[]
    pagination: Pagination
}

export type WantStatus = "active" | "archive" | "closed"
export type Source = "kwork" | "fl" | "upwork"
```

---

## enum — через const object

```ts
// src/core/constants.ts

export const SOURCE = {
    KWORK: "kwork",
    FL: "fl",
    UPWORK: "upwork",
} as const

export type Source = typeof SOURCE[keyof typeof SOURCE]
// Source = "kwork" | "fl" | "upwork"
```

---

## Svelte props — строгая типизация

```svelte
<!-- WantCard.svelte -->
<script lang="ts">
    import type { Want } from "../types"

    // обязательный prop
    export let want: Want

    // опциональный prop с дефолтом
    export let compact: boolean = false
</script>
```

---

## store — типизированный writable

```ts
// src/store/wants.ts
import { writable, derived } from "svelte/store"
import type { Want, Filters, Pagination } from "../types"

interface WantsState {
    wants: Want[]
    filters: Filters
    pagination: Pagination | null
    loading: boolean
    error: string | null
}

const initialState: WantsState = {
    wants: [],
    filters: { source: null, categoryId: null, priceMin: null, priceMax: null, page: 1 },
    pagination: null,
    loading: false,
    error: null,
}

export const wantsStore = writable<WantsState>(initialState)

// derived — вычисляемое значение
export const totalPages = derived(
    wantsStore,
    $store => $store.pagination?.totalPages ?? 0
)
```

---

## api — типизированные функции

```ts
// src/api/wants.ts
import type { Filters, WantsResponse } from "../types"
import { ApiError, NetworkError } from "../core/errors"

export async function fetchWants(filters: Filters): Promise<WantsResponse> {
    const params = new URLSearchParams(
        Object.fromEntries(
            Object.entries(filters).filter(([, v]) => v !== null).map(([k, v]) => [k, String(v)])
        )
    )

    let response: Response
    try {
        response = await fetch(`/api/v1/wants?${params}`)
    } catch {
        throw new NetworkError()
    }

    if (!response.ok) {
        const body = await response.json().catch(() => ({}))
        throw new ApiError(response.status, body.error?.code ?? "UNKNOWN", body.error?.message ?? "Request failed")
    }

    return response.json()
}
```

---

## что запрещено

```ts
// ❌ any
const data: any = await fetch(url)
function process(x: any) { ... }

// ✅ unknown с narrowing
async function fetchRaw(url: string): Promise<unknown> { ... }
if (typeof data === "object" && data !== null && "id" in data) { ... }

// ❌ as без необходимости
const want = data as Want

// ✅ type guard
function isWant(x: unknown): x is Want {
    return typeof x === "object" && x !== null && "id" in x
}

// ❌ non-null assertion без уверенности
const el = document.querySelector(".list")!

// ✅ явная проверка
const el = document.querySelector(".list")
if (!el) throw new Error(".list element not found")
```