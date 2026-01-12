# Serverless Caching (`use cache: remote`)

## When to Use `'use cache: remote'`

The `'use cache: remote'` directive is designed for **Serverless deployment environments** (like Vercel) where in-memory caches don't persist across requests. While regular `'use cache'` works at build time, runtime caching in Serverless environments requires a remote cache handler.

### The Problem with `'use cache'` in Serverless

In Serverless environments:
- **Build-time caching works**: Content prerendered at build time is included in the static shell
- **Runtime in-memory cache doesn't persist**: Each request may hit a different instance, so in-memory cache entries are lost between requests
- **Result**: Every request fetches data again, even with `'use cache'` directive

### The Solution: `'use cache: remote'`

`'use cache: remote'` uses a **remote cache handler** (like KV store, Redis, or platform-provided cache) that persists across Serverless instances:

```typescript
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

// ✅ GOOD: Use 'use cache: remote' for Vercel/Serverless deployments
export async function fetchRS(params: RSFetchParams) {
  'use cache: remote'
  cacheTag(`rs-${params.maPeriod}-${params.threshold}`)
  cacheLife('hours')
  
  // This is cached in a remote handler (KV store) and persists across requests
  const response = await fetchFromBackend('/api/v1/sector-rs/rs', params)
  return response.json()
}
```

## Comparison: `'use cache'` vs `'use cache: remote'`

| Feature | `'use cache'` | `'use cache: remote'` |
|---------|---------------|----------------------|
| **Build-time caching** | ✅ Yes (included in static shell) | ✅ Yes |
| **Runtime caching (self-hosted)** | ✅ Yes (in-memory, persists) | ✅ Yes (remote handler) |
| **Runtime caching (Serverless)** | ❌ No (in-memory, doesn't persist) | ✅ Yes (remote handler, persists) |
| **Cache storage** | In-memory LRU | Remote cache handler (KV, Redis, etc.) |
| **Use case** | Self-hosted deployments, build-time content | Serverless deployments (Vercel, etc.) |

## Decision: When to Use Which

```
Is this deployed on a Serverless platform (Vercel, etc.)?
│
├─► YES → Use 'use cache: remote' for runtime caching
│         (Build-time caching still works with regular 'use cache')
│
└─► NO (self-hosted) → Use 'use cache' (in-memory cache persists)
```

## Example: Vercel Deployment

```typescript
// features/products/queries.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

// For Vercel deployment, use 'use cache: remote'
export async function fetchProducts() {
  'use cache: remote'  // ← Use remote for Serverless
  cacheLife('hours')
  cacheTag('products')
  
  const response = await fetch('https://api.example.com/products')
  if (!response.ok) {
    throw new Error('Failed to fetch products')
  }
  
  return response.json()
}
```

### Important Notes

1. **Vercel automatically provides remote cache handler**: No additional configuration needed when using `'use cache: remote'` on Vercel
2. **Shared across all users**: Remote caches are shared, so only use for non-user-specific data
3. **For user-specific data**: Use `'use cache: private'` instead (client-side caching)
4. **Performance**: Remote cache requires a network roundtrip, but still faster than fetching from origin on every request
