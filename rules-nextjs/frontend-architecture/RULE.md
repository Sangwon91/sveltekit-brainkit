---
alwaysApply: false
---

# Frontend Architecture

Pragmatic Feature-based Architecture for Next.js 16+ (App Router).

---

## Philosophy

**Colocation Over Layering:** Organize by Feature Domain, not file type. Each feature in `@/features/{name}` is a standalone mini-library exposing a public API via `index.ts` (client-safe) and `server.ts` (server-only).

---

## Quick Reference

### Feature Structure

```
features/{name}/
├── components/    # Server Containers + Client Views
├── services/      # Pure business logic (server-only)
├── queries.ts     # Read path ('use cache')
├── actions.ts     # Write path ('use server')
├── parsers.ts     # nuqs URL state parsers
├── types.ts       # All types (re-export shared + define local)
├── index.ts       # ★ Client-safe exports
└── server.ts      # ★ Server-only exports
```

### Key Patterns

```typescript
// ✅ queries.ts - Read Path
import "server-only";
import { cacheTag, cacheLife } from 'next/cache';

export async function getData(id: string) {
  'use cache';
  cacheTag(`data-${id}`);
  cacheLife('minutes');
  return await fetchData(id);
}

// ✅ actions.ts - Write Path
'use server';
import { revalidateTag } from 'next/cache';

export async function updateData(id: string, data: FormData) {
  await saveData(id, data);
  revalidateTag(`data-${id}`);
}
```

### State Priority

1. **URL** → nuqs parsers (`shallow: true`, `limitUrlUpdates: throttle()`)
2. **Server** → `queries.ts` with `'use cache'` (or TanStack Query for complex cases)
3. **Feature Store** → `features/{name}/stores` (Zustand)
4. **Global Store** → `@/stores` (Session, Theme only)

---

## Do's & Don'ts (Cheat Sheet)

| ✅ Do | ❌ Don't |
|-------|----------|
| Use `'use cache'` in queries | Use `'use server'` for GET |
| Export server-only from `server.ts` | Export server-only from `index.ts` |
| Use nuqs parsers for URL state | Parse `searchParams` manually |
| Wrap with `<Suspense>` in page.tsx | Write logic in page.tsx |
| Use `revalidateTag()` for invalidation | Use deprecated `unstable_cache` |

---

## Tech Stack

- **Framework:** Next.js 16+ (App Router, Cache Components)
- **URL State:** nuqs
- **Validation:** Zod
- **Client State:** TanStack Query (when needed), Zustand

---

## Workflow

1. Create feature folder in `features/{name}/`
2. Define types in `types.ts`
3. Implement services in `services/` (with `server-only`)
4. Create queries in `queries.ts` (with `'use cache'`)
5. Create actions in `actions.ts` (with `'use server'`)
6. Build Server Container + Client View components
7. Set up barrel files (`index.ts` + `server.ts`)
8. Compose in `app/` page with `<Suspense>`

---

## References

- See [Directory Structure](references/directory_structure.md) for full tree
- See [Feature Anatomy](references/feature_anatomy.md) for detailed breakdowns
- See [Barrel Files Pattern](references/barrel_files.md) for import/export rules
- See [State Management](references/state_management.md) for hierarchy details
- See [URL State & nuqs](references/url_state_nuqs.md) for parser patterns
- See [Cache Patterns](references/cache_patterns.md) for caching details
- See [Do's and Don'ts](references/dos_and_donts.md) for complete checklist
