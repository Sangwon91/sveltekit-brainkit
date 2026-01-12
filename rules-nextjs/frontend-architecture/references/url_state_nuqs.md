# URL State & nuqs Reference

Comprehensive patterns for type-safe URL state management with nuqs.

---

## Parser Definition

Define parsers in `parsers.ts` within each feature:

```typescript
// features/momentum-analysis/parsers.ts
import { parseAsInteger, parseAsStringEnum } from 'nuqs/server'
import { createSearchParamsCache } from 'nuqs/server'

// Parser definitions (shared between server and client)
export const momentumParsers = {
  lookbackDays: parseAsInteger.withDefault(20),
  topN: parseAsInteger.withDefault(20),
  chartType: parseAsStringEnum(['bump', 'line', 'bar']).withDefault('bump'),
  sortBy: parseAsStringEnum(['rank', 'momentum', 'name']).withDefault('rank'),
} as const

// Server cache creation
export const momentumCache = createSearchParamsCache(momentumParsers)

// Type extraction
export type MomentumParams = {
  lookbackDays: number
  topN: number
  chartType: 'bump' | 'line' | 'bar'
  sortBy: 'rank' | 'momentum' | 'name'
}
```

---

## Server Component Usage

### Step 1: Parse in Page Component

```typescript
// app/(app)/dashboard/momentum-analysis/page.tsx
import { momentumCache } from '@/features/momentum-analysis/parsers'
import { type SearchParams } from 'nuqs/server'

export default async function Page({
  searchParams,
}: {
  searchParams: Promise<SearchParams>
}) {
  // ⚠️ REQUIRED: Initialize cache with parse()
  await momentumCache.parse(searchParams)
  return <MomentumChartContainer />
}
```

### Step 2: Access in Nested Server Components

```typescript
// features/momentum-analysis/components/MomentumChartContainer.tsx
import { momentumCache } from '../parsers'
import { fetchMomentum } from '../queries'

export async function MomentumChartContainer() {
  // Access cached values in nested components
  const params = momentumCache.all()
  // Or individual: const lookbackDays = momentumCache.get('lookbackDays')
  
  const momentumData = await fetchMomentum(params)
  return <BumpChart data={momentumData} />
}
```

---

## Client Component Usage

```typescript
// features/momentum-analysis/components/MomentumControls.tsx
'use client'

import { useQueryStates, throttle } from 'nuqs'
import { momentumParsers } from '../parsers' // Same parsers as server

export function MomentumControls() {
  const [params, setParams] = useQueryStates(momentumParsers, {
    limitUrlUpdates: throttle(300), // Throttle frequent updates
    shallow: true,                   // Prevent server re-renders (default)
  })
  
  return (
    <div>
      <input
        type="number"
        value={params.lookbackDays}
        onChange={(e) => setParams({ lookbackDays: Number(e.target.value) })}
      />
    </div>
  )
}
```

---

## Common Parsers Location

Shared parsers in `lib/nuqs/parsers.ts`:

```typescript
// lib/nuqs/parsers.ts
import { parseAsInteger, parseAsString } from 'nuqs'

export const commonParsers = {
  page: parseAsInteger.withDefault(1),
  pageSize: parseAsInteger.withDefault(20),
  search: parseAsString.withDefault(''),
  sort: parseAsString.withDefault(''),
} as const
```

---

## Key Options

| Option | Purpose |
|--------|---------|
| `shallow: true` | Prevent server re-renders (default) |
| `limitUrlUpdates: throttle(300)` | Throttle updates (for sliders) |
| `limitUrlUpdates: debounce(500)` | Debounce updates (for search inputs) |

> **Deprecated:** `throttleMs` and `debounceMs` - use `limitUrlUpdates` instead
