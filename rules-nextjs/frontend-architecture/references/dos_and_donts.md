# Do's and Don'ts Reference

Complete checklist for Frontend Architecture compliance.

---

## ✅ Do's

### Architecture
- Use **dual barrel files** (`index.ts` + `server.ts`) for every feature
- Keep `page.tsx` as a thin Composition Shell
- Wrap Server Component Containers with `<Suspense>` in `page.tsx`

### Server/Client Boundary
- Use `import "server-only"` in `queries.ts` and `services/`
- Export server-only modules from `server.ts` only
- Export client-safe modules from `index.ts`

### Data Fetching
- Use `'use cache'` directive with `cacheTag()` and `cacheLife()` in `queries.ts`
- Use `revalidateTag()` or `updateTag()` in `actions.ts` for cache invalidation
- Use **Zod** for all input validation in `actions.ts`

### Types
- Centralize all types (including shared) in `types.ts`
- Import types via relative path (`../types`)

### URL State (nuqs)
- Use nuqs parsers for type-safe URL state in `parsers.ts`
- Export parsers from Feature's `index.ts`
- Use `createSearchParamsCache` → `cache.parse()` → `cache.get()` in Server Components
- Use `useQueryState`/`useQueryStates` with `shallow: true` in Client Components
- Apply throttling/debouncing via `limitUrlUpdates: throttle()` or `debounce()`

---

## ❌ Don'ts

### Data Fetching
- DON'T use `useEffect` to fetch initial data - use Server Components
- DON'T use Server Actions (`'use server'`) for GET operations - use `queries.ts`
- DON'T put business logic in Route Handlers (`app/api`) unless building public REST API
- DON'T use deprecated `cache` (React) or `unstable_cache` - use `'use cache'`

### Server/Client Boundary
- DON'T export `server-only` dependent modules from `index.ts`
- DON'T write data fetching logic directly in `page.tsx`

### Types
- DON'T import types directly from `@/types` in feature files - re-export from `types.ts` first

### URL State (nuqs)
- DON'T parse `searchParams` manually - use nuqs parsers
- DON'T use `useQueryState` in Server Components - use `createSearchParamsCache`
- DON'T use `parseServerClient` - deprecated, use `createSearchParamsCache`
- DON'T use `throttleMs` or `debounceMs` - deprecated, use `limitUrlUpdates`
- DON'T update URL in Server Components - only in Client Components
- DON'T forget `shallow: true` to prevent server re-renders
- DON'T update URL without throttling/debouncing for frequent updates
