# Cache Patterns Reference

SvelteKit uses standard HTTP caching headers for controlling response caching.

---

## Setting Cache Headers

In your `load` functions (`+page.server.ts`), use `setHeaders` to control caching behavior (CDN, Browser).

```typescript
// src/routes/blog/[slug]/+page.server.ts
export async function load({ setHeaders, params }) {
  const post = await db.getPost(params.slug);

  setHeaders({
    // Cache in CDN for 1 hour, browser for 5 minutes
    'cache-control': 'public, max-age=300, s-maxage=3600'
  });

  return { post };
}
```

---

## Cache Profiles (Recommended)

Define shared constants for consistency.

```typescript
// src/lib/server/cache.ts
export const CACHE_CONTROL = {
  REALTIME: 'no-store',
  SHORT: 'public, max-age=60, s-maxage=60',        // 1 minute
  MEDIUM: 'public, max-age=300, s-maxage=3600',    // 1 hour
  LONG: 'public, max-age=86400, s-maxage=604800',   // 1 week
};
```

Usage:
```typescript
setHeaders({ 'cache-control': CACHE_CONTROL.MEDIUM });
```

---

## Invalidation (Client-Side)

SvelteKit provides mechanisms to re-run `load` functions without a full page reload.

- `invalidate(url)`: Re-runs any `load` function that depends on this URL.
- `invalidate('custom:tag')`: Re-runs `load` functions that explicitly `depends('custom:tag')`.
- `invalidateAll()`: Re-runs everything.

Example:
```typescript
// src/routes/+page.svelte
import { invalidate } from '$app/navigation';

async function refresh() {
  await invalidate('app:dashboard');
}

// src/routes/+page.server.ts
export async function load({ depends }) {
  depends('app:dashboard');
  return { data: ... };
}
```
