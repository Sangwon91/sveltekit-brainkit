# Cache Components & Caching Strategy

## Core Principle
Next.js 16+ introduces **Cache Components** (`'use cache'`), enabling **Partial Prerendering (PPR)**. You must explicit declare caching intent.

## The `'use cache'` Directive
Caches the return value of an async function or component.

```typescript
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

export async function getProducts() {
  'use cache'
  cacheLife('hours')       // Time-based expiration
  cacheTag('products')     // Tag-based invalidation
  return db.product.findMany()
}
```

## Cache Profiles (`cacheLife`)
| Profile | Use Case |
|---------|----------|
| `seconds` | Real-time (stock prices) |
| `minutes` | Feeds / Comments |
| `hours` | Site content (most common) |
| `days` | Static blog posts |
| `weeks` | Documentation |
| `max` | Permanent data |

## Serverless Deployment (Vercel)
**CRITICAL**: In Serverless environments (like Vercel), in-memory cache does not persist.
-   **Use `'use cache: remote'`** to use the platform's remote cache (KV/Redis).
-   Regular `'use cache'` only works for build-time static generation in Serverless.

```typescript
export async function getData() {
  'use cache: remote' // Persists across serverless functions
  cacheLife('hours')
  // ...
}
```

## Invalidation
-   `revalidateTag('tag')`: Mark as stale (background update).
-   `updateTag('tag')`: Purge immediately (immediate update).
