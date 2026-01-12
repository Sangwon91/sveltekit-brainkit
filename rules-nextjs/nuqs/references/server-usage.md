# Server-Side Usage (Read-Only)

In Next.js App Router, Server Components are the consumers of URL state. `nuqs` provides `createSearchParamsCache` to make accessing these parameters typesafe and deep-accessible (avoiding prop drilling).

## Pattern: The Cache Object

### 1. Define Cache
Define the cache alongside your parsers.

```typescript
// features/dashboard/parsers.ts
import { parseAsInteger, createSearchParamsCache } from 'nuqs/server'

export const parsers = {
  tab: parseAsInteger.withDefault(0)
}
export const cache = createSearchParamsCache(parsers)
```

### 2. Parse in Page (Entry Point)
You MUST call `cache.parse(searchParams)` in the `page.tsx` before any component tries to read from the cache.

```typescript
// app/dashboard/page.tsx
import { cache } from '@/features/dashboard/parsers'

type Props = {
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}

export default async function Page({ searchParams }: Props) {
  // 🟢 CRITICAL: Initialize cache
  await cache.parse(searchParams)

  return (
    <Suspense fallback={<Skeleton />}>
      <DashboardContent />
    </Suspense>
  )
}
```

### 3. Consume in Components
Any Server Component nested inside the Page can now read the cache directly.

```typescript
// features/dashboard/components/DashboardContent.tsx
import { cache } from '../parsers'

export async function DashboardContent() {
  // Read value (type-safe)
  const tabIndex = cache.get('tab') 
  
  // Or read all parameters
  const { tab } = cache.all()

  // Use for data fetching
  const data = await db.getData({ tab: tabIndex })
  
  return <RenderData data={data} />
}
```

## Important Rules

1.  **Read-Only**: Server Components cannot "set" URL params. Changing URL requires a navigation (Client Component).
2.  **No Prop Drilling**: The `cache` object acts like a Server-Side Context store for search params.
3.  **Type Safety**: `cache.get()` returns the typed value (e.g., `number`), not a string.
