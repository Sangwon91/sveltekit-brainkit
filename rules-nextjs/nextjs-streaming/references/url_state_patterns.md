---
title: URL State Patterns with nuqs
description: Integrating nuqs with streaming for type-safe URL state.
---

# URL State Patterns (nuqs)

Next.js streaming works best when components read their own parameters from the URL. Use **nuqs** for type-safety.

## Core Principles
1.  **Define Parsers:** Define parsers in `parsers.ts`.
2.  **Create Cache:** Use `createSearchParamsCache` for Server Components.
3.  **Parse in Page:** Call `cache.parse(searchParams)` in the Page component.
4.  **Read in Component:** Use `cache.get()` or `cache.all()` in the deep data-fetching component.

## Implementation Pattern

### 1. Define Parsers
```typescript
// features/analysis/parsers.ts
import { parseAsInteger, createSearchParamsCache } from 'nuqs/server';

export const parsers = {
  days: parseAsInteger.withDefault(7),
  limit: parseAsInteger.withDefault(10),
};
export const analysisCache = createSearchParamsCache(parsers);
```

### 2. Parse & Stream
```typescript
// app/analysis/page.tsx
import { analysisCache } from '@/features/analysis/parsers';
import { ChartContainer } from '@/features/analysis/server';

export default async function Page({ searchParams }) {
  // 1. Parse once at page level
  await analysisCache.parse(searchParams);

  return (
    // 2. Stream components
    <Suspense fallback={<Skeleton />}>
      <ChartContainer /> {/* No props passed! */}
    </Suspense>
  );
}
```

### 3. Fetch in Component
```typescript
// features/analysis/components/ChartContainer.tsx
import { analysisCache } from '../parsers';

export async function ChartContainer() {
  // 3. Read directly from cache deep in the tree
  const { days, limit } = analysisCache.all();
  
  const data = await fetchData({ days, limit });
  return <Chart data={data} />;
}
```

## Client Updates
Use `shallow: true` (default) in `useQueryStates` to update URL without triggering a full server render if only client-side filtering involves. However, for data-dependent changes (like pagination) where you need new server data, `shallow: true` still avoids full page reload in Next.js App Router, as it updates URL and lets the Server Component re-run if needed via `refresh`, OR more commonly in App Router, `shallow: false` (or executing a navigation) triggers the Server Component refresh.

*Note: In nuqs, `shallow: true` updates browser history without running loaders/server components. `shallow: false` triggers server component re-execution.*

For streaming data updates (e.g. changing filter -> new data), use `shallow: false` or standard navigation to trigger server re-fetch.
