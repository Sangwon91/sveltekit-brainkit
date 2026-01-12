---
trigger: always_on
glob: "*"
description: "Frontend Architecture Rules"
---

# Frontend Architecture

Pragmatic Feature-based Architecture for SvelteKit v5.

---

## Philosophy

**Colocation Over Layering:** Organize by Feature Domain in `$lib/features/{name}`. Each feature is a standalone module with clear server/client separation via `.server.ts` convention.

---

## Quick Reference

### Feature Structure

```
src/lib/features/{name}/
├── components/       # Reusable UI Components
├── state.svelte.ts   # ★ Client State (Runes, 함수형)
├── api.server.ts     # ★ Business Logic (Server Only)
└── types.ts          # Shared Types
```

### Key Patterns

```typescript
// ✅ api.server.ts (Service Layer)
import { db } from '$lib/server';
export async function getData(id: string) {
  return await db.query(id);
}

// ✅ +page.server.ts (Data Loading)
import { getData } from '$lib/features/feature/api.server';

export async function load({ setHeaders }) {
  setHeaders({ 'cache-control': 'max-age=60' });
  return { data: await getData('123') };
}

// ✅ state.svelte.ts (Client State - 함수형)
export function createFeatureState(initialCount = 0) {
  let count = $state(initialCount);
  const double = $derived(count * 2);
  
  return {
    get count() { return count; },
    get double() { return double; },
    increment() { count += 1; }
  };
}
```

### State Priority

1. **URL** → `$page.url.searchParams` (native)
2. **Server** → `PageData` (from `load`)
3. **Feature State** → Runes in `state.svelte.ts`
4. **Global State** → Shared Runes / Context

---

## Do's & Don'ts (Cheat Sheet)

| ✅ Do | ❌ Don't |
|-------|----------|
| Use `setHeaders` for caching | Use `cache-control` meta tags |
| Use `.server.ts` suffix for server code | Import server modules in components |
| Use Runes (`$state`, `$derived`) | Fetch data in `onMount` for SSR data |
| Use `+page.server.ts` for data | Mix business logic in `+page.svelte` |

---

## Tech Stack

- **Framework:** SvelteKit v5
- **State:** Svelte 5 Runes ($state, $derived)
- **URL State:** Native (`$page.url`, `goto`)
- **Validation:** Zod / Valibot
- **Styling:** TailwindCSS / shadcn-svelte

---

## Workflow

1. Create feature folder in `$lib/features/{name}/`
2. Define types in `types.ts`
3. Implement logic in `api.server.ts` (database/business logic)
4. Implement client state in `state.svelte.ts` (if needed)
5. Build UI components in `components/`
6. Compose in `routes/` using `+page.server.ts` (load) and `+page.svelte` (render)

---

## References

- See [Directory Structure](frontend-architecture/references/directory_structure.md) for full tree
- See [Feature Anatomy](frontend-architecture/references/feature_anatomy.md) for detailed breakdowns
- See [State Management](frontend-architecture/references/state_management.md) for hierarchy details
- See [URL State](frontend-architecture/references/url_state.md) for deep linking patterns
- See [Cache Patterns](frontend-architecture/references/cache_patterns.md) for HTTP caching
- See [Do's and Don'ts](frontend-architecture/references/dos_and_donts.md) for complete checklist
