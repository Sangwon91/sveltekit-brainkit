---
title: Component Granularity & Data Colocation
description: Patterns for component-level data fetching and request deduplication.
---

# Component Granularity: "Who Needs It, Fetches It"

## The Golden Rule

**Fetch data directly in the component that renders it.** Avoid fetching data in a parent component and passing it down as props, unless absolutely necessary.

### Why This Matters

1.  **Independent Suspense Boundaries:** Enables parallel streaming.
2.  **Static Shell Maximization:** Allows granular caching with `'use cache'`.
3.  **Better Error Boundaries:** Isolates errors to specific components.
4.  **Code Clarity:** Co-locates data requirements with UI.

### Anti-Pattern: Parent Fetching

```typescript
// ❌ BAD: Parent fetches all data (Sequential Loading)
export async function DashboardPage() {
  const [user, orders] = await Promise.all([fetchUser(), fetchOrders()]);
  return (
    <div>
      <UserProfile user={user} />
      <OrderList orders={orders} />
    </div>
  );
}
```

### Correct Pattern: Component Fetching

```typescript
// ✅ GOOD: Components fetch their own data (Parallel Streaming)
export async function UserProfileContainer() {
  const user = await fetchUser();
  return <UserProfile user={user} />;
}

export async function OrderListContainer() {
  const orders = await fetchOrders();
  return <OrderList orders={orders} />;
}

export default function DashboardPage() {
  return (
    <div className="grid gap-4">
      <Suspense fallback={<UserSkeleton />}>
        <UserProfileContainer />
      </Suspense>
      <Suspense fallback={<OrderSkeleton />}>
        <OrderListContainer />
      </Suspense>
    </div>
  );
}
```

## Request Deduplication

When multiple components fetch the same data, you must ensure requests are deduplicated.

### 1. `fetch()` (Automatic)
Next.js extends `fetch` to automatically memoize requests within a render pass.

```typescript
// ✅ Automatic deduplication
async function getData() {
  const res = await fetch('https://api.example.com/data');
  return res.json();
}
```

### 2. `react.cache()` (Manual)
For ORM calls or DB queries, wrap the function with `react.cache()` to deduplicate within a request.

```typescript
// ✅ Deduplicated per request
import { cache } from 'react';
import { db } from '@/lib/db';

export const getUser = cache(async (id: string) => {
  return await db.user.findUnique({ where: { id } });
});
```

### 3. `'use cache'` (Persistent)
To cache across multiple requests (shared cache) and include in the Static Shell.

```typescript
// ✅ Persisted cache (Build time + Runtime)
export async function getUser(id: string) {
  'use cache';
  cacheLife('minutes');
  return await db.user.findUnique({ where: { id } });
}
```

## Exceptions to the Rule

Only break this rule if:
1.  **Shared Heavy Computation:** Multiple components need the exact same expensive calculation result.
2.  **Tight Coupling:** Components are visually and logically inseparable.
3.  **Auth Context:** Needed globally (though usually handled via Context or `layout.tsx`).
