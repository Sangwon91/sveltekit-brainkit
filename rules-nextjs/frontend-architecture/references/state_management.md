# State Management Reference

Follow this priority order for managing state.

---

## State Hierarchy (Priority Order)

### 1. URL (Query Params) - Shareable State

For filters, pagination, tabs, and any state that should be bookmarkable/shareable.

**Use:** nuqs with type-safe parsers

| Context | API |
|---------|-----|
| Server Components | `createSearchParamsCache` → `cache.parse()` → `cache.get()` |
| Client Components | `useQueryState` / `useQueryStates` with `shallow: true` |

**Benefits:**
- Type-safe via parsers
- Prevents server re-renders with `shallow: true` (default)
- Supports throttling/debouncing via `limitUrlUpdates`

See [URL State & nuqs](url_state_nuqs.md) for detailed patterns.

---

### 2. Server State - Database Data

**Default Approach:** Use `queries.ts` with `'use cache'` directive.

**When to use TanStack Query instead:**

| Scenario | Use |
|----------|-----|
| Static/Infrequently changing data | `queries.ts` |
| Large datasets (>1MB) | TanStack Query |
| Frequent filter/sort changes | TanStack Query + nuqs |
| Real-time updates | TanStack Query |
| Complex interdependent queries | TanStack Query |

**TanStack Query + nuqs Pattern:**

```typescript
'use client'

import { useQuery } from '@tanstack/react-query'
import { useQueryStates, throttle } from 'nuqs'
import { analysisParsers } from '../parsers'

export function AnalysisDashboard() {
  // 1. URL state management (nuqs)
  const [filters, setFilters] = useQueryStates(analysisParsers, {
    shallow: true,
    limitUrlUpdates: throttle(300),
  })

  // 2. Client-side data fetching (TanStack Query)
  const { data, isLoading, error } = useQuery({
    queryKey: ['analysis', filters],
    queryFn: () => fetchAnalysisData(filters),
    staleTime: 30000,
  })

  if (isLoading) return <Skeleton />
  if (error) return <ErrorDisplay error={error} />
  
  return <AnalysisChart data={data} filters={filters} onFilterChange={setFilters} />
}
```

---

### 3. Local Feature Store - Complex UI State

For complex UI state specific to one feature.

**Location:** `features/{name}/stores`

**Use:** Zustand or similar

---

### 4. Global Store - App-Wide State

Only for truly global, app-wide data.

**Location:** `@/stores`

**Examples:** User Session, Theme, Toast notifications

---

## Cross-Feature Dependencies

- **Rule:** Avoid direct imports between `features/feature-a` and `features/feature-b`
- **Solution:** Lift shared state to `app/page.tsx` (via URL or server fetch) and pass as props
