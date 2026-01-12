---
alwaysApply: false
---

# nuqs (Next.js URL State) Rules

## Philosophy
**URL as Single Source of Truth.**
Manage shareable state (filters, pagination) in the URL to enable bookmarking and deep linking. client-side updates should use `shallow: true` to prevent unnecessary server re-renders, while maintaining type safety via `nuqs` parsers.

## Core Principles
-   **Type Safety**: Always use defined parsers (`parseAsInteger`, etc.) to read/write URL params.
-   **Client-Driven**: Updates happen on the Client (`useQueryState`). Server simply reads (`createSearchParamsCache`).
-   **Performance**: Use `shallow: true` (default) to update URL without hitting the server. Use `throttle` for frequent inputs.
-   **Server Read-Only**: NEVER try to update URL from a Server Component.

## Tech Stack
-   **Library**: `nuqs`
-   **Version**: 2.0+ (App Router compatible)
-   **Context**: Next.js 16+ App Router

## Implementation Checklist
-   [ ] **Setup**: Wrap root layout in `<NuqsAdapter>`.
-   [ ] **Parsers**: Define shared parsers in `parsers.ts` (or per-feature).
-   [ ] **Server**: Use `createSearchParamsCache` for Server Components. Call `parse(searchParams)` in Page.
-   [ ] **Client**: Use `useQueryState` / `useQueryStates` for interactive components.
-   [ ] **Optimization**: Add `limitUrlUpdates: throttle(ms)` for sliders/search inputs.

## Quick Reference

### 1. Setup
```tsx
// app/layout.tsx
import { NuqsAdapter } from 'nuqs/adapters/next/app'
export default function RootLayout({ children }) {
  return <NuqsAdapter>{children}</NuqsAdapter>
}
```

### 2. Define Parsers & Cache
```ts
// features/search/parsers.ts
import { parseAsInteger, parseAsString, createSearchParamsCache } from 'nuqs/server'

export const searchParsers = {
  q: parseAsString.withDefault(''),
  page: parseAsInteger.withDefault(1),
}
export const searchCache = createSearchParamsCache(searchParsers)
```

### 3. Server Component (Read-Only)
```tsx
// app/search/page.tsx
import { searchCache } from '@/features/search/parsers'

export default async function Page({ searchParams }) {
  // 1. Parse once in Page
  await searchCache.parse(searchParams)
  return <Results />
}

// components/Results.tsx (Server Component)
export async function Results() {
  // 2. Access deeply nested without props drilling
  const { q } = searchCache.all()
  return <div>Results for {q}</div>
}
```

### 4. Client Component (Update)
```tsx
'use client'
import { useQueryStates, throttle } from 'nuqs'
import { searchParsers } from '../parsers'

export function SearchControls() {
  const [params, setParams] = useQueryStates(searchParsers, {
    shallow: true, // No server re-render
    limitUrlUpdates: throttle(300)
  })
  
  return <input value={params.q} onChange={e => setParams({ q: e.target.value })} />
}
```

## References
-   See [Installation & Setup](references/setup.md) for initial configuration.
-   See [Type-Safe Parsers](references/parsers.md) for parser definitions and custom types.
-   See [Server-Side Patterns](references/server-usage.md) for `createSearchParamsCache` and reading state.
-   See [Client-Side Patterns](references/client-usage.md) for `useQueryState`, usage and hooks.
-   See [Performance & Shallow Routing](references/performance.md) for `shallow` options and re-rendering strategies.
