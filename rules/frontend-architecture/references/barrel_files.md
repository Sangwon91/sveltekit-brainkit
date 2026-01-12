# Barrel Files Reference

Features expose **two entry points** to handle the server/client boundary safely in SvelteKit.

---

## `index.ts` - Client-Safe Exports

Exports modules that can be safely imported from **any context** (Client-side components, `+page.svelte`, utilities).

```typescript
// src/lib/features/dashboard/index.ts
/** Dashboard Feature - Client-safe API */

// Components
export { default as DashboardView } from "./components/DashboardView.svelte";

// Client Logic / State
export { DashboardState } from "./logic.svelte";

// Types (always safe)
export type { DashboardData } from "./types";

// NOTE: Server-only exports (api.server.ts) are excluded here.
```

---

## `server.ts` - Server-Only Exports

Exports modules that depend on server-side logic (databases, private environment variables). SvelteKit (via Vite) prevents importing files ending in `.server.ts` into client-side code, so we use this convention for our public server API.

```typescript
// src/lib/features/dashboard/server.ts
/** Dashboard Feature - Server-only API */

// Business Logic / Data Access
export * as dashboardApi from "./api.server";

// We re-export the entire api.server.ts namespace or specific functions
// Usage: import { dashboardApi } from '$lib/features/dashboard/server';
```

---

## Import Patterns

```typescript
// ✅ From Client Components / +page.svelte
import { DashboardView, DashboardState } from "$lib/features/dashboard";

// ✅ From Server Files (+page.server.ts, +layout.server.ts)
import { dashboardApi } from "$lib/features/dashboard/server";

// ❌ WRONG: Will cause build error or runtime failure
// In a Client Component:
import { dashboardApi } from "$lib/features/dashboard/server"; 
// Error: Cannot import .server.ts module in client code.
```
