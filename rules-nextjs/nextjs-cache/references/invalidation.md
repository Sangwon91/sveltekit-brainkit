# Cache Invalidation

## `revalidateTag()` - Stale-While-Revalidate

Marks cache entries as stale. Next request serves stale data while revalidating in background:

```typescript
'use server'
import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: ProductData) {
  await db.product.update({ where: { id }, data })
  
  // Mark 'products' cache as stale
  // Next request: serves stale, revalidates in background
  revalidateTag('products')
}
```

## `updateTag()` - Immediate Refresh

Expires cache immediately and refreshes within the same request:

```typescript
'use server'
import { updateTag } from 'next/cache'

export async function addToCart(productId: string) {
  await db.cart.add({ productId })
  
  // Immediately expire and refresh cart cache
  // Same request sees updated data
  updateTag('cart')
}
```

## Comparison Table

| Function | Behavior | Use When |
|----------|----------|----------|
| `revalidateTag()` | Stale-while-revalidate | Content changes can tolerate brief staleness (blog posts, product catalog) |
| `updateTag()` | Immediate expiry + refresh | User expects immediate feedback (cart, user profile) |

## Tagging Strategy

```typescript
// queries.ts
export async function getProduct(id: string) {
  'use cache'
  cacheTag('products')           // General tag
  cacheTag(`product-${id}`)      // Specific tag
  cacheLife('hours')
  
  return await db.product.findUnique({ where: { id } })
}

// actions.ts
'use server'
import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: ProductData) {
  await db.product.update({ where: { id }, data })
  
  // Invalidate both general and specific caches
  revalidateTag('products')
  revalidateTag(`product-${id}`)
}
```
