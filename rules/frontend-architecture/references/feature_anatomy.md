# Feature Anatomy Reference

Detailed breakdown of the `src/lib/features/{feature-name}/` folder structure.

---

## A. Components (`/components`)

Contains reusable UI components for the feature.

- **Role:** View/Presentation logic.
- **Context:** Runs on both Server (SSR) and Client (Hydration).
- **State:** Use props for data, or `state.svelte.ts` for complex local state.

```svelte
<!-- src/lib/features/dashboard/components/DashboardView.svelte -->
<script lang="ts">
  import type { DashboardData } from '../types';
  import { createDashboardState } from '../state.svelte';

  let { data }: { data: DashboardData } = $props();
  const state = createDashboardState();
</script>

<div>
  <!-- UI utilizing data and state -->
</div>
```

---

## B. API & Business Logic (`api.server.ts`)

- **Role:** Pure business logic and Data Access Layer (DAL).
- **Context:** **Server-Only**. Contains database calls, ORM usage.
- **Naming Pattern:** `*.server.ts` suffix ensures this code never leaks to the client bundle.

```typescript
// src/lib/features/dashboard/api.server.ts
import { db } from '$lib/server/db';
import type { DashboardData } from './types';

export async function getDashboardData(userId: string): Promise<DashboardData> {
  // Direct DB access allowed here
  return await db.query(...);
}

export async function updateSettings(userId: string, settings: any) {
  // Mutation logic
}
```

---

## C. Data Loading (Connector)

SvelteKit uses `load` functions in `+page.server.ts` as the bridge between URL/Request and Feature Logic.

- **Role:** Fetch data using `api.server.ts`, return `PageData`.

```typescript
// src/routes/dashboard/+page.server.ts
import { getDashboardData } from '$lib/features/dashboard/api.server';

export async function load({ locals }) {
  const data = await getDashboardData(locals.user.id);
  return { data };
}
```

---

## D. Actions (Write Path)

SvelteKit uses `actions` in `+page.server.ts` for form submissions and mutations.

```typescript
// src/routes/dashboard/+page.server.ts
import { updateSettings } from '$lib/features/dashboard/api.server';
import { fail } from '@sveltejs/kit';

export const actions = {
  update: async ({ request, locals }) => {
    const data = await request.formData();
    // Validate
    try {
      await updateSettings(locals.user.id, data);
    } catch (e) {
      return fail(400, { error: 'Update failed' });
    }
  }
};
```

---

## E. Client Logic & State (`state.svelte.ts`)

- **Role:** Complex client-side state management using Svelte 5 Runes.
- **Pattern:** 함수형 (Function-based) - JavaScript 생태계 표준

```typescript
// src/lib/features/dashboard/state.svelte.ts
export function createDashboardState(initialFilter = 'all') {
  let filter = $state(initialFilter);
  const label = $derived(`Current filter: ${filter}`);
  
  function setFilter(newFilter: string) {
    filter = newFilter;
  }
  
  return {
    get filter() { return filter; },
    get label() { return label; },
    setFilter
  };
}
```

**왜 함수형인가?**
- React Hooks, Vue Composition API와 동일한 패턴
- 인스턴스 관리가 자동 (Class는 `new` 필요)
- SSR에서 request 간 상태 오염 방지

---

## F. Types (`types.ts`)

- **Role:** Central hub for feature types.
- **Rule:** Export interfaces used by both `api.server.ts` (return types) and components (prop types).
