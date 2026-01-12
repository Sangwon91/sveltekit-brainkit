# Performance & Shallow Routing

One of the biggest performance bottlenecks in Next.js App Router applications is unintentional Server Component re-renders. `nuqs` helps control this via the `shallow` option.

## Understanding `shallow: true` (Default)

When you update a URL parameter with `shallow: true` (which is the default in `nuqs`):

1.  **URL updates** in the browser address bar.
2.  **History API** is updated (push or replace).
3.  **No Network Request** is sent to the Next.js server.
4.  **Client Components** using `useQueryState` re-render with the new value.
5.  **Server Components DO NOT re-render**. They retain their current state.

### When to use `shallow: true`
-   Filtering a list that is already fully loaded on the client.
-   Updating UI state (tabs, view modes, toggles).
-   Searching/Filtering when data is fetched via Client-Side Query (e.g., TanStack Query).
-   Complex interactive dashboards where server round-trips are too slow.

```typescript
// ✅ Good: Shallow update for client-side chart toggle
const [view, setView] = useQueryState('view', { shallow: true })
// URL changes, but Server Comp does not rerun logic
```

## Understanding `shallow: false`

When `shallow: false` is set:

1.  Next.js triggers a **Server Action / Navigation**.
2.  The server receives the new URL parameters.
3.  **Server Components Re-execute**.
4.  `cache.parse(searchParams)` gets the NEW values.
5.  The new UI payload (RSC Payload) is streamed back to the client.

### When to use `shallow: false`
-   Pagination (where you need to fetch the next page from the DB).
-   Search where results are server-rendered.
-   Changing a filter that affects a server-side DB query.

```typescript
// ✅ Good: Trigger server fetch for new page
const [page, setPage] = useQueryState('page', { 
  shallow: false,
  history: 'push' 
})
```

## Hybrid Performance Pattern

For optimal performance in data-heavy apps, combine **Client-Side Fetching** (TanStack Query) with **URL State** (nuqs).

1.  **Server Comp**: Renders initial skeleton/layout. Reads initial params ONLY for SEO/OG tags if needed.
2.  **URL State**: Managed by `nuqs` with `shallow: true`.
3.  **Data Fetching**: `useQuery` listens to the URL state variables as keys.

This avoids the "Server Component Rerender Loop" entirely while keeping the URL as the source of truth.

```typescript
function Dashboard() {
  // 1. URL State (Client only, no server trip)
  const [filter] = useQueryState('filter', { shallow: true })

  // 2. Data Fetching (reacts to filter change)
  const { data } = useQuery({
    queryKey: ['data', filter],
    queryFn: () => fetchData(filter)
  })

  return <Chart data={data} />
}
```
