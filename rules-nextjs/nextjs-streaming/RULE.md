---
alwaysApply: false
---

# Next.js Streaming & PPR Rules

## Core Philosophy
- **Component Granularity:** One Data Source = One Component = One Suspense Boundary.
- **Data Colocation:** Components fetch their own data. Avoid parent-level fetching (`Promise.all`) unless data is interdependent.
- **Parallel Streaming:** Place Suspense boundaries close to data sources to enable independent, parallel loading.
- **Static Shell Maximization:** Use `'use cache'` to include cacheable content in the initial static shell (PPR).

## Core Knowledge (The Cheat Sheet)

- **The Golden Rule:** "Who Needs It, Fetches It". Don't prop-drill server data.
- **Request Deduplication:**
  - `fetch()`: Automatically memoized in Next.js.
  - `react.cache()`: Use for ORM/DB calls within a request.
  - `'use cache'`: Use for persistent caching (cross-request).
- **URL Pattern:** Use **nuqs** `createSearchParamsCache` in Server Components to read URL state deep in the tree without prop drilling.
- **Suspense Placement:** Wrap *each* async container in its own `<Suspense>`.
- **Loading UI:** Use Skeletons that match the finalized layout to prevent layout shift.

### Code Snippets

**✅ Good: Parallel Streaming**
```typescript
export default function Page() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}><Header /></Suspense>
      <div className="grid grid-cols-2">
        <Suspense fallback={<ChartSkeleton />}><ChartContainer /></Suspense>
        <Suspense fallback={<FeedSkeleton />}><FeedContainer /></Suspense>
      </div>
    </>
  )
}
```

**❌ Bad: Sequential Blocking**
```typescript
export default async function Page() {
  // ❌ Fetches all at once, blocks entire page until slowest completes
  const [chart, feed] = await Promise.all([getChart(), getFeed()]); 
  return <Dashboard chart={chart} feed={feed} />
}
```

## Tech Stack
- **Next.js 16+** (App Router, PPR)
- **React 19** (Suspense, `use` hook)
- **nuqs** (URL State Management)

## Workflow
1.  **Identify Data Sources**: Break down UI into sections requiring independent data.
2.  **Create Containers**: Create `*Container.tsx` (Server Component) for each section.
3.  **Implement Fetching**: Call data functions directly in Containers. Use `react.cache` or `'use cache'` appropriately.
4.  **Handle URL State**: Use `nuqs` cache parser in `page.tsx`, read params in Containers.
5.  **Add Suspense**: Wrap Containers in `page.tsx` with specific Skeletons.
6.  **Verify**: Check build output for `◐` (Partial) symbol to confirm PPR.

## References
- See [Component Granularity & Colocation](references/component_granularity.md)
- See [Suspense Boundary Patterns](references/suspense_boundaries.md)
- See [URL State & Streaming](references/url_state_patterns.md)
- See [Maximizing Static Shell](references/static_shell.md)
- See [Refactoring Examples](references/refactoring_examples.md)