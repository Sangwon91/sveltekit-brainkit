# URL State & Data Fetching

## Core Principle
Use **nuqs** to sync URL search params with component state. The server reads the URL to fetch initial data; the client updates the URL to trigger changes.

## Server-Side Reading
Use `createSearchParamsCache` to define parsers and read params in Server Components.

```typescript
// parsers.ts
import { parseAsInteger } from 'nuqs/server'
export const parsers = { page: parseAsInteger.withDefault(1) }
export const cache = createSearchParamsCache(parsers)

// page.tsx (Server Component)
export default async function Page({ searchParams }) {
  await cache.parse(searchParams) // Parse once
  const { page } = cache.all()    // Access typesafe
  const data = await fetchData(page)
  // ...
}
```

## Client-Side Writing
Use `useQueryState` or `useQueryStates` to update URL without full page reload.
```typescript
'use client'
const [page, setPage] = useQueryState('page', parsers.page)
```

## Optimization
-   **Shallow Routing**: Default in nuqs (`shallow: true`). Updates URL without re-running Server Components (good for client-side filtering).
-   **Deep Routing**: Set `shallow: false` to re-trigger Server Component data fetching (good for pagination).
