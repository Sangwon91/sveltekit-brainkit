# Streaming & Suspense

## Dynamic Rendering Triggers

When Next.js encounters work that can't complete during prerendering (network requests, DB queries, etc.), you must explicitly handle it:

| Strategy | Directive | Result |
|----------|-----------|--------|
| **Cache it** | `'use cache'` | Cached result included in static shell |
| **Stream it** | `<Suspense>` | Fallback in shell, content streams at request time |

> **Critical:** If content is neither cached nor wrapped in Suspense, you'll see: `Uncached data was accessed outside of <Suspense>` error.

### Complete List of APIs That Require Explicit Handling

When these are used, you must wrap with `<Suspense>` or use `'use cache'`:

| API | Import | Can Use `use cache`? |
|-----|--------|---------------------|
| `cookies()` | `next/headers` | ❌ No - requires request context |
| `headers()` | `next/headers` | ❌ No - requires request context |
| `searchParams` | Page props | ❌ No - requires request context |
| `params` (without `generateStaticParams`) | Page props | ❌ No - requires request context |
| `connection()` | `next/server` | ❌ No - explicitly defers to request |
| `fetch()` (network request) | global | ✅ Yes - cacheable |
| `db.query()` (database) | ORM | ✅ Yes - cacheable |
| `Math.random()`, `Date.now()` | global | ✅ Yes - if same value for all users is OK |

## Streaming with Suspense

Use `<Suspense>` when:
1. Content requires **runtime data** (cookies, headers, searchParams)
2. Content must be **always fresh** (no caching acceptable)
3. Content has **user-specific personalization** that can't be parameterized

### Basic Pattern

```typescript
import { Suspense } from 'react'
import { cookies } from 'next/headers'

async function PersonalizedContent() {
  const cookieStore = await cookies()
  const userId = cookieStore.get('userId')?.value
  
  // This runs at request time, streams to client
  const recommendations = await getRecommendations(userId)
  
  return <RecommendationList items={recommendations} />
}

export default function Page() {
  return (
    <>
      <h1>Welcome</h1>  {/* Static shell */}
      
      <Suspense fallback={<RecommendationSkeleton />}>
        <PersonalizedContent />  {/* Streams at request time */}
      </Suspense>
    </>
  )
}
```

### Parallel Streaming for Better Performance

Place Suspense boundaries close to dynamic content to maximize static shell and enable parallel streaming:

```typescript
export default function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-4">
      {/* These stream in parallel, not sequentially */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />
      </Suspense>
      
      <Suspense fallback={<TableSkeleton />}>
        <RecentOrders />
      </Suspense>
      
      <Suspense fallback={<ListSkeleton />}>
        <TopProducts />
      </Suspense>
      
      <Suspense fallback={<StatsSkeleton />}>
        <UserStats />
      </Suspense>
    </div>
  )
}
```

### Suspense Boundary Placement Strategy

```
❌ BAD: Single boundary (sequential loading)
┌─────────────────────────────────────┐
│ <Suspense>                          │
│   <A /> → <B /> → <C /> → <D />     │  All wait for slowest
│ </Suspense>                         │
└─────────────────────────────────────┘

✅ GOOD: Multiple boundaries (parallel loading)
┌─────────────────────────────────────┐
│ <Suspense><A /></Suspense>          │  Each loads independently
│ <Suspense><B /></Suspense>          │
│ <Suspense><C /></Suspense>          │
│ <Suspense><D /></Suspense>          │
└─────────────────────────────────────┘
```

### Using `connection()` to Explicitly Defer

When you need request-time execution without accessing runtime APIs:

```typescript
import { connection } from 'next/server'
import { Suspense } from 'react'

async function UniqueContent() {
  // Explicitly defer to request time
  await connection()
  
  // Now these run per-request
  const random = Math.random()
  const now = Date.now()
  
  return <div>Random: {random}, Time: {now}</div>
}
```
