---
alwaysApply: false
---

# Next.js 16+ Cache Components & PPR Rules

## Philosophy
- **Explicit Caching**: Always use `'use cache'`, `cacheLife()`, and `cacheTag()` to declare caching intent.
- **Maximize Static Shell**: Cache everything possible to include it in the initial static HTML payload (PPR).
- **Strategic Streaming**: Use `<Suspense>` only for genuinely dynamic, user-specific, or request-time data.

## Core Concepts & Patterns

### 1. Enabling Cache Components
Ensure `experimental.ppr` (or `cacheComponents` in newer versions) is enabled in `next.config.ts`.
See [Cache Components & Prerendering](references/cache-components.md) for setup details.

### 2. The `use cache` Directive
Use `'use cache'` to cache async functions or components. This creates a static entry at build time and a persistent cache entry at runtime.

```typescript
import { cacheLife, cacheTag } from 'next/cache'

async function getProducts() {
  'use cache'
  cacheLife('hours')
  cacheTag('products')
  return db.product.findMany()
}
```

- **Runtime Data**: You CANNOT access `cookies()`, `headers()`, or `searchParams` directly inside `'use cache'`. Extract values and pass them as arguments.
- **Serverless**: Use `'use cache: remote'` for Vercel/Serverless to persist cache across requests.
See [Serverless Caching](references/serverless.md).

### 3. Streaming with Suspense
Wrap data that *cannot* be cached (personalized, request-time) in `<Suspense>`.
See [Streaming & Suspense](references/streaming.md).

```typescript
<Suspense fallback={<Skeleton />}>
  <UserPreferences /> {/* Accesses cookies/headers */}
</Suspense>
```

### 4. Cache Invalidation
Use tags for granular invalidation.
See [Cache Invalidation](references/invalidation.md).

```typescript
import { revalidateTag } from 'next/cache'
await updateData()
revalidateTag('products') 
```

### 5. Integration with Client Cache (TanStack Query)
Layer your caching: Server Cache for origin protection, Client Cache for UX.
See [Client-Side Integration](references/client-side-integration.md).

### 6. URL State & Caching
Combine `nuqs` parsers with `cacheTag` to cache based on URL parameter combinations.
See [URL State Caching](references/url-state-caching.md).

## DOs and DON'Ts

- **DO** use granular `<Suspense>` boundaries to allow parallel streaming.
- **DO** use `updateTag()` for immediate feedback on user actions.
- **DO** check build output for `◐` (Partial Prerendering) symbol.
- **DON'T** use `export const dynamic = 'force-dynamic'` (It's dead).
- **DON'T** wrap the entire page in one `<Suspense>`.
- **DON'T** use `unstable_cache`. Use `'use cache'` instead.

## Tech Stack
- Next.js 16+ (App Router)
- React 19

## References
- [Cache Components & Prerendering](references/cache-components.md)
- [Serverless Caching (Remote)](references/serverless.md)
- [Streaming & Suspense](references/streaming.md)
- [Cache Invalidation](references/invalidation.md)
- [Migration Guide](references/migration.md)
- [URL State Caching (nuqs)](references/url-state-caching.md)
- [Client-Side Integration](references/client-side-integration.md)