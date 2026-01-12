# Feature Anatomy Reference

Detailed breakdown of the `src/lib/features/{feature-name}/` folder structure.

---

## A. Components (`/components`)

Contains reusable UI components for the feature.

- **Role:** View/Presentation logic.
- **Context:** Runs on both Server (SSR) and Client (Hydration).
- **State:** Use props for data, or `logic.svelte.ts` for complex local state.

```svelte
<!-- src/lib/features/dashboard/components/DashboardView.svelte -->
<script lang="ts">
  import type { DashboardData } from '../types';
  import { DashboardState } from '../logic.svelte';

  let { data }: { data: DashboardData } = $props();
  const state = new DashboardState();
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
import { dashboardApi } from '$lib/features/dashboard/server';

export async function load({ locals }) {
  const data = await dashboardApi.getDashboardData(locals.user.id);
  return { data };
}
```

---

## D. Actions (Write Path)

SvelteKit uses `actions` in `+page.server.ts` for form submissions and mutations.

```typescript
// src/routes/dashboard/+page.server.ts
import { dashboardApi } from '$lib/features/dashboard/server';
import { fail } from '@sveltejs/kit';

export const actions = {
  update: async ({ request, locals }) => {
    const data = await request.formData();
    // Validate
    try {
      await dashboardApi.updateSettings(locals.user.id, data);
    } catch (e) {
      return fail(400, { error: 'Update failed' });
    }
  }
};
```

---

## E. Client Logic & State (`logic.svelte.ts`)

- **Role:** Complex client-side state management using Svelte 5 Runes.
- **Replaces:** `useState`, `useEffect`, complex stores.

```typescript
// src/lib/features/dashboard/logic.svelte.ts
export class DashboardState {
  filter = $state('all');
  
  // Derived state with Runes
  label = $derived(`Current filter: ${this.filter}`);

  setFilter(newFilter: string) {
    this.filter = newFilter;
  }
}
```

---

## F. Types (`types.ts`)

- **Role:** Central hub for feature types.
- **Rule:** Export interfaces used by both `api.server.ts` (return types) and components (prop types).
