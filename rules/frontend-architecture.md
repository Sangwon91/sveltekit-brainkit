---
trigger: always_on
glob: "*"
description: "Frontend Architecture Rules"
---

# Frontend Architecture

Pragmatic Feature-based Architecture for SvelteKit v5.

---

## Philosophy

**Colocation Over Layering:** Organize by Feature Domain in `$lib/features/{name}`. Each feature is a standalone module exposing a public API via `index.ts` (client) and `server.ts` (server).

---

## Quick Reference

### Feature Structure

```
src/lib/features/{name}/
├── components/    # Reusable UI Components
├── logic.svelte.ts # ★ Client State (Runes)
├── api.server.ts  # ★ Business Logic (Server Only)
├── types.ts       # Shared Types
├── index.ts       # Client Public API
└── server.ts      # Server Public API
```

### Key Patterns

```typescript
// ✅ api.server.ts (Service Layer)
import { db } from '$lib/server';
export async function getData(id: string) {
  return await db.query(id);
}

// ✅ +page.server.ts (Data Loading)
import { featureApi } from '$lib/features/feature/server';

export async function load({ setHeaders }) {
  setHeaders({ 'cache-control': 'max-age=60' });
  return { data: await featureApi.getData('123') };
}

// ✅ logic.svelte.ts (Client State)
export class FeatureState {
  count = $state(0);
  double = $derived(this.count * 2);
}
```

### State Priority

1. **URL** → `page.url` / `sveltekit-search-params`
2. **Server** → `PageData` (from `load`)
3. **Feature State** → Runes in `logic.svelte.ts`
4. **Global State** → Shared Runes / Context

---

## Do's & Don'ts (Cheat Sheet)

| ✅ Do | ❌ Don't |
|-------|----------|
| Use `setHeaders` for caching | Use `cache-control` meta tags |
| Export server API from `server.ts` | Import `.server.ts` in components |
| Use Runes (`$state`) | Use complex `writable` stores for local state |
| Use `+page.server.ts` for data | Fetch data in `onMount` |

---

## Tech Stack

- **Framework:** SvelteKit v5
- **State:** Svelte 5 Runes ($state, $derived)
- **URL State:** sveltekit-search-params
- **Validation:** Zod / Valibot
- **Styling:** TailwindCSS / shadcn-svelte

---

## Workflow

1. Create feature folder in `$lib/features/{name}/`
2. Define types in `types.ts`
3. Implement logic in `api.server.ts` (database/business logic)
4. Implement client state in `logic.svelte.ts` (if needed)
5. Build UI components in `components/`
6. Set up barrel files (`index.ts` + `server.ts`)
7. Compose in `routes/` using `+page.server.ts` (load) and `+page.svelte` (render)

---

## References

- See [Directory Structure](frontend-architecture/references/directory_structure.md) for full tree
- See [Feature Anatomy](frontend-architecture/references/feature_anatomy.md) for detailed breakdowns
- See [Barrel Files Pattern](frontend-architecture/references/barrel_files.md) for import/export rules
- See [State Management](frontend-architecture/references/state_management.md) for hierarchy details
- See [URL State](frontend-architecture/references/url_state.md) for deep linking patterns
- See [Cache Patterns](frontend-architecture/references/cache_patterns.md) for HTTP caching
- See [Do's and Don'ts](frontend-architecture/references/dos_and_donts.md) for complete checklist
