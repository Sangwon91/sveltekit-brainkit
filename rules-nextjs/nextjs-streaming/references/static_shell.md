---
title: Maximizing Static Shell
description: Using 'use cache' and cache tags to maximize static shell coverage.
---

# Maximizing Static Shell

The **Static Shell** is the part of the page that is prerendered at build time. Maximizing this improves TTFB and FCP.

## Strategy
1.  **Identify Cacheable Data:** Data not dependent on request-specific info (like cookies/headers) or highly dynamic URL params can often be cached.
2.  **Use 'use cache':** Explicitly mark data fetching functions to be cached.
3.  **Separate Dynamic Parts:** Isolate request-dependent parts into their own components wrapped in Suspense.

## Example

```typescript
// ✅ Mixed Static and Dynamic
export default function ProductPage() {
  return (
    <>
      {/* 1. Static Shell (Header, Nav, etc.) - Prerendered */}
      <Header />
      
      {/* 2. Cached Dynamic Content - Included in Shell via PPR */}
      <ProductCatalog /> 
      
      {/* 3. True Dynamic Content - Streamed */}
      <Suspense fallback={<PersonalizedSkeleton />}>
        <UserRecommendations />
      </Suspense>
    </>
  );
}
```

## Caching Implementation

```typescript
// features/products/queries.ts
export async function getProductCatalog() {
  'use cache';
  cacheLife('hours');
  cacheTag('catalog');
  // ... fetch logic
}
```

## Building Verification
Run `pnpm build` to verify PPR status:
- `○` (Static): Fully static.
- `◐` (Partial): Static shell + Streaming.
- `λ` (Dynamic): Fully dynamic (avoid if possible).
