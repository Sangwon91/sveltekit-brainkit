# Cache Components & Prerendering

## What is Cache Components?

Cache Components (enabled via `cacheComponents: true`) introduces **Partial Prerendering (PPR)** - a rendering model that produces a **static HTML shell** at build time, with dynamic content streaming in at request time.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Static Shell                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Header    │  │    Nav      │  │   Cached Blog Posts     │ │
│  │  (static)   │  │  (static)   │  │   (use cache)           │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Loading...  ← Suspense fallback (in static shell)         ││
│  │  ─────────────────────────────────────────────────────────  ││
│  │  UserPreferences streams in at request time                 ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Enabling Cache Components

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}

export default nextConfig
```

## Automatic Prerendering

Operations that complete during prerendering are automatically added to the static shell:

```typescript
// These are automatically prerendered - no configuration needed
import fs from 'node:fs'

export default async function Page() {
  // ✅ Synchronous file system read
  const config = fs.readFileSync('./config.json', 'utf-8')
  
  // ✅ Module imports
  const constants = await import('./constants.json')
  
  // ✅ Pure computations
  const processed = JSON.parse(config).items.map(item => item.value * 2)
  
  return (
    <div>
      <h1>{constants.appName}</h1>
      <ul>
        {processed.map((value, i) => <li key={i}>{value}</li>)}
      </ul>
    </div>
  )
}
```

### Verification

Check build output to verify prerendering:

```
Route (app)                              Size     First Load JS
┌ ○ /                                    5.2 kB         89 kB    ← Fully static
├ ◐ /blog                                3.1 kB         87 kB    ← PPR (partial)
└ λ /dashboard                           4.5 kB         88 kB    ← Dynamic

○  (Static)   fully prerendered as static HTML
◐  (Partial)  partially prerendered (PPR) - has streaming content
λ  (Dynamic)  server-rendered on demand
```

## Caching with `use cache`

The `'use cache'` directive caches async function/component return values:

```typescript
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

// Function-level caching
export async function getProducts() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')
  
  const products = await db.product.findMany()
  return products
}

// Component-level caching
async function ProductList() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')
  
  const products = await db.product.findMany()
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  )
}
```

### Cache Profiles Reference (`cacheLife`)

| Profile | Stale | Revalidate | Expire | Use Case |
|---------|-------|------------|--------|----------|
| `'seconds'` | 0 | 1s | 60s | Near real-time (stock prices) |
| `'minutes'` | 0 | 60s | 300s | Frequently updated (feeds) |
| `'hours'` | 0 | 3600s | 86400s | Multiple daily updates |
| `'days'` | 0 | 86400s | 604800s | Daily updates (blog posts) |
| `'weeks'` | 0 | 604800s | 2592000s | Weekly updates |
| `'max'` | 0 | 2592000s | ∞ | Rarely changes |

### Custom Cache Configuration

```typescript
import { cacheLife } from 'next/cache'

async function getAnalytics() {
  'use cache'
  cacheLife({
    stale: 3600,      // Serve stale for 1 hour
    revalidate: 7200, // Revalidate after 2 hours
    expire: 86400,    // Expire after 1 day
  })
  
  return await fetchAnalyticsData()
}
```

### Cache Keys (Automatic)

Arguments and closed-over values automatically become part of the cache key:

```typescript
// Each unique (userId, filter) combination gets its own cache entry
async function getUserOrders(userId: string, filter: string) {
  'use cache'
  cacheLife('minutes')
  cacheTag(`orders-${userId}`)
  
  return await db.order.findMany({ 
    where: { userId, status: filter } 
  })
}
```

### Using with Runtime Data (Extract Values Pattern)

Runtime data cannot be used directly in `'use cache'` scope, but you can extract values and pass them:

```typescript
import { cookies } from 'next/headers'
import { Suspense } from 'react'

export default function Page() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <ProfileContent />
    </Suspense>
  )
}

// Non-cached: reads runtime data
async function ProfileContent() {
  const session = (await cookies()).get('session')?.value
  
  // Pass extracted value to cached function
  return <CachedUserData sessionId={session} />
}

// Cached: receives value as prop (becomes cache key)
async function CachedUserData({ sessionId }: { sessionId: string }) {
  'use cache'
  cacheLife('minutes')
  cacheTag(`user-${sessionId}`)
  
  const userData = await fetchUserData(sessionId)
  return <div>{userData.name}</div>
}
```
