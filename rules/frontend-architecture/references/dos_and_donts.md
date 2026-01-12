# Do's and Don'ts Reference

Complete checklist for Frontend Architecture compliance in SvelteKit v5.

---

## ✅ Do's

### Architecture
- **Do** follow the `state.svelte.ts` pattern for complex client state (함수형 Runes).
- **Do** use `+page.server.ts` as the primary data fetching layer.
- **Do** import directly from feature modules (e.g., `from '$lib/features/dashboard/api.server'`).

### Server/Client Boundary
- **Do** put server-only logic in `.server.ts` files or `$lib/server/` directory.
- **Do** use the `*.server.ts` naming convention for server-only modules.

### Data Fetching
- **Do** use `setHeaders` for HTTP caching.
- **Do** use `invalidate` / `applyAction` for UI updates after mutations.
- **Do** use `Zod` or similar validation in Form Actions.

### State
- **Do** use native URL APIs (`$page.url`, `goto`) for shareable state.
- **Do** use `$state()` Runes for local component state.
- **Do** use `$derived()` for computed values.

---

## ❌ Don'ts

### Data Fetching
- **Don't** fetch data in `onMount` if it can be loaded server-side.
- **Don't** perform direct database queries in `+page.svelte`.
- **Don't** use `load` functions for expensive mutations - use Form Actions.

### Server/Client Boundary
- **Don't** import `.server.ts` files into client components.
- **Don't** leak sensitive environment variables to the client (`$env/static/private`).

### State
- **Don't** use global state for local feature state - use `state.svelte.ts`.
- **Don't** sync derived state manually - use `$derived()`.
- **Don't** create class instances for simple state - use 함수형 패턴.
