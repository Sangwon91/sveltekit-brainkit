---
alwaysApply: false
---

# Client-Side Data Fetching

**Mantra:** "Server First, Client when Necessary."

## Philosophy
- **Hybrid & Pragmatic:** Prefer Server Components by default. Use Client Fetching only for large datasets (>1MB), frequent user interactions (filters/sorts), or real-time updates.
- **Sync URL & Server:** The URL (via `nuqs`) drives the State (via `TanStack Query`).

## Tech Stack
- **@tanstack/react-query**: State Management & Caching.
- **nuqs**: Type-safe URL State Management (Search Params).
- **axios**: HTTP Client with interceptors.

## Core Patterns

### 1. The Standard Integration Flow
The robust way to build search/filter interfaces.

1.  **Define Parsers** (`parsers.ts`): Define strict types for URL params.
2.  **API Route** (`route.ts`): Validate using `zod`, fetch data.
3.  **Query Options** (`queries.ts`): Create reusable `queryOptions`.
4.  **Component** (`page.tsx`): Wire `useQueryStates` -> `useQuery`.

```typescript
// Component Wiring Example
const [filters] = useQueryStates(analysisParsers) // 1. Support URL
const { data } = useQuery(analysisQueryOptions(filters)) // 2. Fetch Data
```

### 2. When to use Client Fetching?
- **Filtering/Sorting**: Dashboards with complex controls.
- **Infinite Scroll**: "Load More" patterns.
- **Real-time**: Polling or Live data.
- **Large Data**: Avoid serializing 5MB+ JSON from Server Components.

## Workflow
- [ ] **Setup Provider**: Ensure `QueryClientProvider` implementation separates Server/Client clients.
- [ ] **Configure Axios**: Setup global error handling and auth tokens.
- [ ] **Define Types**: Create `parsers.ts` for all URL parameters.
- [ ] **Create Query**: Implement `queryOptions` factory for type-safety.
- [ ] **Implement View**: Connect `nuqs` state to `useQuery`.

## References
- See [Setup & Configuration](references/setup.md) for Provider and Axios setup.
- See [Integration Guide](references/integration-guide.md) for the complete nuqs + Query pattern.
- See [Advanced Patterns](references/advanced-patterns.md) for Dependent Queries and Optimistic Updates.
- See [Best Practices](references/best-practices.md) for Keys, StaleTime, and Performance.
- See [Next.js Cache Integration](references/nextjs-cache-integration.md) for 'use cache' rules.
