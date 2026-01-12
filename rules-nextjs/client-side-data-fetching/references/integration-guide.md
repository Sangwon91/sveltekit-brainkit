# Client-Side Data Fetching: Integration Guide

The core pattern combines **Server State** (TanStack Query) with **URL State** (nuqs) to create shareable, cacheable, and high-performance applications.

## End-to-End Flow

### 1. Define Parsers (Source of Truth)

Centralize your type definitions using `nuqs`.

```typescript
// features/analysis/parsers.ts
import { parseAsInteger, parseAsStringEnum, parseAsIsoDateTime } from 'nuqs/server'

export const analysisParsers = {
  startDate: parseAsIsoDateTime,
  metric: parseAsStringEnum(['revenue', 'users']).withDefault('revenue'),
  page: parseAsInteger.withDefault(1),
}
export type AnalysisFilters = typeof analysisParsers
```

### 2. Create Type-Safe API Route

Use `zod` to validate incoming requests in your Route Handler.

```typescript
// app/api/analysis/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'

const schema = z.object({
  metric: z.enum(['revenue', 'users']),
  page: z.coerce.number(),
})

export async function GET(request: NextRequest) {
  const searchParams = Object.fromEntries(request.nextUrl.searchParams)
  const validated = schema.safeParse(searchParams)
  
  if (!validated.success) return NextResponse.json({ error: 'Invalid' }, { status: 400 })
  
  const data = await db.query(validated.data)
  return NextResponse.json(data)
}
```

### 3. Query Function & Options

Create a reusable query options factory. This ensures the Query Key is always consistent with the fetch parameters.

```typescript
// features/analysis/queries.ts
import { queryOptions } from '@tanstack/react-query'
import apiClient from '@/lib/api/client'
import type { AnalysisFilters } from './parsers'

const fetchAnalysis = async (filters: AnalysisFilters) => {
  const { data } = await apiClient.get('/analysis', { params: filters })
  return data
}

export const analysisQueryOptions = (filters: AnalysisFilters) => queryOptions({
  queryKey: ['analysis', filters], // Filters automatically part of key
  queryFn: () => fetchAnalysis(filters),
})
```

### 4. Client Component (The Wiring)

Wire it all together. URL changes trigger re-renders, which updates the Query Key, which triggers a refetch.

```typescript
// features/analysis/components/Dashboard.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { useQueryStates, throttle } from 'nuqs'
import { analysisParsers } from '../parsers'
import { analysisQueryOptions } from '../queries'

export function Dashboard() {
  // 1. URL State
  const [filters, setFilters] = useQueryStates(analysisParsers, {
    shallow: true,
    limitUrlUpdates: throttle(300),
  })

  // 2. Server State (Syncs with filters)
  const { data, isFetching } = useQuery(
    analysisQueryOptions(filters)
  )

  return (
    <>
      <FilterBar filters={filters} onChange={setFilters} />
      {isFetching && <Spinner />}
      <Chart data={data} />
    </>
  )
}
```
