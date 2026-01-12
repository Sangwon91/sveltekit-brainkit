# Cache Patterns Reference

Next.js 16 Cache Components patterns for data fetching.

---

## Configuration

Enable Cache Components in `next.config.ts`:

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true,
}

export default nextConfig
```

---

## Basic Pattern

Use `'use cache'` directive with `cacheTag` and `cacheLife`:

```typescript
import "server-only";
import { cacheTag, cacheLife } from 'next/cache';
import { fetchStockInternal } from './services/stock-service';
import type { StockChartData } from '../types';

export async function getStockChartData(ticker: string): Promise<StockChartData> {
  'use cache';
  cacheTag(`stock-${ticker}`);
  cacheLife('minutes');
  return await fetchStockInternal(ticker);
}
```

---

## Cache Profiles

| Profile | Revalidate | Use Case |
|---------|------------|----------|
| `seconds` | 1 second | Real-time data (stock prices) |
| `minutes` | 1 minute | Frequently updated (feeds) |
| `hours` | 1 hour | Multiple daily updates |
| `days` | 1 day | Daily updates (blog posts) |
| `weeks` | 1 week | Weekly updates |
| `max` | 30 days | Rarely changes |

---

## Cache Invalidation

Use `revalidateTag` or `updateTag` in Server Actions:

```typescript
'use server';
import { revalidateTag } from 'next/cache';

export async function updateProductAction(id: string, data: ProductData) {
  await updateProduct(id, data);
  revalidateTag(`product-${id}`);  // Invalidate specific cache
  revalidateTag('products');        // Invalidate list cache
}
```

### `revalidateTag` vs `updateTag`

| Function | Behavior |
|----------|----------|
| `revalidateTag` | Marks stale, revalidates on next request |
| `updateTag` | Immediate refresh within same request |

---

## Important Notes

- `cacheTag` is more granular than `revalidatePath`
- Match tags in `queries.ts` with invalidation in `actions.ts`
- **Don't use:** deprecated `cache` (React) or `unstable_cache`
