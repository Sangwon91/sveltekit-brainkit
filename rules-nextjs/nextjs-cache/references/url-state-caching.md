# URL State & Caching Strategy

## Strategies for Caching with URL State

URL state-dependent data can be cached, but entries are created per parameter combination. Use `nuqs` parsers to read URL state and include parameters in the cache key.

## Pattern: Integration with `nuqs` Parsers

```typescript
// features/momentum-analysis/parsers.ts
import { parseAsInteger, parseAsStringEnum } from 'nuqs/server'
import { createSearchParamsCache } from 'nuqs/server'

// 1. Define parsers (shared between server & client)
export const momentumParsers = {
  lookbackDays: parseAsInteger.withDefault(20),
  topN: parseAsInteger.withDefault(20),
  chartType: parseAsStringEnum(['bump', 'line', 'bar']).withDefault('bump'),
} as const

// 2. Create server-side cache
export const momentumCache = createSearchParamsCache(momentumParsers)

export type MomentumParams = {
  lookbackDays: number
  topN: number
  chartType: 'bump' | 'line' | 'bar'
}
```

```typescript
// features/momentum-analysis/queries.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import type { MomentumParams } from '../parsers'

export async function fetchMomentum(params: MomentumParams) {
  'use cache: remote'
  cacheLife('hours')
  
  // 3. Include generic tag AND specific parameter tag
  cacheTag('momentum')
  cacheTag(`momentum-${params.lookbackDays}-${params.topN}-${params.chartType}`)
  
  return await fetchFromBackend('/api/v1/momentum', params)
}
```

## Using in Page & Components

```typescript
// app/(app)/dashboard/momentum-analysis/page.tsx
import { Suspense } from 'react'
import { momentumCache } from '@/features/momentum-analysis/parsers'
import { MomentumChartContainer } from '@/features/momentum-analysis/server'

export default async function MomentumAnalysisPage({
  searchParams,
}: {
  searchParams: Promise<SearchParams>
}) {
  // ⚠️ MUST call parse to initialize cache
  await momentumCache.parse(searchParams)
  
  return (
    <Suspense fallback={<ChartSkeleton />}>
      <MomentumChartContainer />
    </Suspense>
  )
}

// features/momentum-analysis/components/MomentumChartContainer.tsx
import { momentumCache } from '../parsers'
import { fetchMomentum } from '../queries'

export async function MomentumChartContainer() {
  // Access cache in nested server component
  const params = momentumCache.all()
  const momentumData = await fetchMomentum(params)
  
  return <BumpChart data={momentumData} />
}
```

## Invalidation Strategy

When URL state changes or data updates, invalidate only relevant cache entries:

```typescript
// features/momentum-analysis/actions.ts
'use server'
import { revalidateTag } from 'next/cache'

export async function refreshMomentumData(params: MomentumParams) {
  // Invalidate specific parameter combination
  revalidateTag(`momentum-${params.lookbackDays}-${params.topN}-${params.chartType}`)
}

export async function refreshAllMomentumData() {
  // Invalidate all momentum data
  revalidateTag('momentum')
}
```

## Preventing Server Re-renders

Use `nuqs`'s `shallow: true` to prevent server re-renders on URL updates, while still updating the URL history.

```typescript
// Client Component
import { useQueryStates, throttle } from 'nuqs'

const [params, setParams] = useQueryStates(momentumParsers, {
  limitUrlUpdates: throttle(300),
  shallow: true, // Prevents server re-render
})
```

**Important:** `shallow: true` prevents the Server Components from re-executing. If your data depends on URL state (like above), you often WANT server re-execution to fetch new data, so you might need `shallow: false` (default for some updates) or handle data fetching on the client side with TanStack Query if frequent updates are needed.
