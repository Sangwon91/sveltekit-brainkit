# Feature Anatomy Reference

Detailed breakdown of the `features/{feature-name}/` folder structure.

---

## A. Components (`/components`)

Contains both Server and Client components:

### Server Component Container

Async component that fetches data via `queries.ts` and passes to Client Components.

```typescript
// features/dashboard/components/DashboardContent.tsx
import { DashboardView } from "./DashboardView";
import { fetchDashboardData } from "../queries";

export async function DashboardContent() {
  const data = await fetchDashboardData();
  return <DashboardView data={data} />;
}
```

> **Export Rule:** Server Containers import `server-only` modules → export via `server.ts`

### Client Component (View/Presentational)

```typescript
// features/dashboard/components/DashboardView.tsx
"use client";

export function DashboardView({ data }: DashboardViewProps) {
  const [filter, setFilter] = useState("all");
  return <div>...</div>;
}
```

> **Export Rule:** Client Components are safe → export via `index.ts`

---

## B. Services (`/services`) - The "Brain"

- **Role:** Pure business logic and Data Access Layer (DAL)
- **Constraint:** Must include `import 'server-only'` at the top
- **Usage:** Called by both `queries.ts` (Read) and `actions.ts` (Write)
- **Dependencies:** No React knowledge. Just DB calls and logic.

---

## C. Queries (`queries.ts`) - The "Read" Path

- **Context:** Next.js 16 Server Components Data Fetching with Cache Components
- **Role:** Async functions called by Server Component Containers
- **Constraint:** Must include `import 'server-only'` at the top. **NEVER** use `'use server'`

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

### Cache Profiles Reference

| Profile | Use Case | Revalidate |
|---------|----------|------------|
| `seconds` | Real-time data (stock prices) | 1 second |
| `minutes` | Frequently updated (feeds) | 1 minute |
| `hours` | Multiple daily updates | 1 hour |
| `days` | Daily updates (blog posts) | 1 day |
| `weeks` | Weekly updates | 1 week |
| `max` | Rarely changes | 30 days |

---

## D. Actions (`actions.ts`) - The "Write" Path

- **Context:** Server Actions for Mutations (Create, Update, Delete)
- **Constraint:** Must start with `'use server'`

**Workflow:**
1. Validate Input (Zod)
2. Verify Auth/Permissions
3. Call Service logic
4. Invalidate Cache by Tag (`revalidateTag` or `updateTag`)

```typescript
'use server';
import { z } from 'zod';
import { revalidateTag } from 'next/cache';
import { createOrderService } from './services/order-service';
import type { CreateOrderInput } from '../types';

const schema = z.object({
  productId: z.string(),
  quantity: z.number().min(1),
});

export async function createOrderAction(formData: FormData) {
  const parsed = schema.parse(Object.fromEntries(formData)) as CreateOrderInput;
  await createOrderService(parsed);      // 1. Logic
  revalidateTag('orders');               // 2. Invalidate cache by tag
}
```

---

## E. Types (`types.ts`) - Type Import Convention

- **Role:** Central hub for all Feature types
- **Purpose:** Feature dependencies visible in one place

**Rules:**
1. Re-export shared types from `@/types`, `@/lib/types`
2. Feature files import via relative path `../types`
3. Define feature-specific types here

```typescript
// features/order/types.ts

// Re-export shared types (document external dependencies)
export type { User } from '@/types/user';
export type { ApiResponse, PaginatedResult } from '@/lib/types';

// Feature-specific types
export interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  status: OrderStatus;
  createdAt: Date;
}

export type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered';

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}
```

---

## F. Parsers (`parsers.ts`) - URL State

See [URL State & nuqs](url_state_nuqs.md) for detailed patterns.
