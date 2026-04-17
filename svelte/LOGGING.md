# logging conventions — svelte / typescript

соглашения по логированию для svelte frontend leadar.
в браузере нет JSON-логов — используем `console.*` с префиксами.

---

## уровни

| метод | когда |
|---|---|
| `console.error` | ошибки в catch блоках, unhandled rejection |
| `console.warn` | нежелательные ситуации (пустой ответ, retry) |
| `console.info` | только в dev: ключевые события (авторизация, загрузка) |
| `console.debug` | только в dev: промежуточные данные |

`console.log` — **запрещён** в финальном коде.
`console.info` / `console.debug` — только за флагом `import.meta.env.DEV`.

---

## формат — префикс модуля

```ts
// префикс — название store или модуля в квадратных скобках
console.error("[loadWants]", e)
console.error("[fetchWants]", "HTTP", response.status, response.url)
console.warn("[filtersStore]", "empty response for filters", filters)
```

---

## паттерны

### ошибки в store actions

```ts
export async function loadWants(): Promise<void> {
    try {
        const data = await fetchWants(get(filtersStore))
        wantsStore.update(s => ({ ...s, wants: data.data }))
    } catch (e) {
        wantsStore.update(s => ({ ...s, error: resolveError(e) }))
        console.error("[loadWants]", e)  // логируем оригинальную ошибку
    }
}
```

### dev-only логи

```ts
if (import.meta.env.DEV) {
    console.debug("[wantsStore] loaded", data.data.length, "wants")
}
```

### глобальный обработчик в main.ts

```ts
window.addEventListener("unhandledrejection", e => {
    console.error("[unhandledrejection]", e.reason)
    e.preventDefault()
})
```

---

## что запрещено

```ts
// ❌ console.log
console.log("data", data)

// ❌ логируем без контекста
console.error(e)  // непонятно откуда

// ❌ технические детали пользователю — только в console, не в UI
wantsStore.update(s => ({ ...s, error: e.message }))  // "Failed to fetch"

// ✅ в UI — понятный текст, в console — оригинальная ошибка
wantsStore.update(s => ({ ...s, error: "Нет соединения с сервером" }))
console.error("[loadWants]", e)
```
