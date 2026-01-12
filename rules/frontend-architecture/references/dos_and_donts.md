# Do's and Don'ts Reference

Complete checklist for Frontend Architecture compliance in SvelteKit v5.

---

## ✅ Do's

### Architecture
- **Do** follow the `logic.svelte.ts` pattern for complex client state (Runes).
- **Do** use `+page.server.ts` as the primary data fetching layer ("Backends for Frontends").

### Server/Client Boundary
- **Do** put server-only logic in `.server.ts` files or `$lib/server/` directory.
- **Do** export client-safe modules from `index.ts`.
- **Do** export server-only API from `server.ts`.

### Data Fetching
- **Do** use `setHeaders` for HTTP caching.
- **Do** use `invalidate` / `applyAction` for UI updates after mutations.
- **Do** use `Zod` or similar validation in Form Actions.

### State
- **Do** use URL search params for shareable state (filters, pagination).
- **Do** use `$state()` runes for local component state.

---

## ❌ Don'ts

### Data Fetching
- **Don't** fetch data in `onMount` (`useEffect`) if it can be loaded server-side.
- **Don't** perform direct database queries in `+page.svelte` (not possible, but don't try to shim it via `static` imports).
- **Don't** use `load` functions for expensive mutations - use Form Actions.

### Server/Client Boundary
- **Don't** import `.server.ts` files into client components.
- **Don't** leak sensitive environment variables to the client (`$env/static/private`).

### State
- **Don't** use global stores for local feature state - use a class with Runes in `logic.svelte.ts`.
- **Don't** sync derived state manually - use `$derived()`.
