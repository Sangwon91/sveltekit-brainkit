# State Management Reference

Follow this priority order for managing state in SvelteKit v5.

---

## State Hierarchy (Priority Order)

### 1. URL (Query Params) - Shareable State

For filters, pagination, tabs, and any state that should be bookmarkable/shareable.

**Use:** `sveltekit-search-params` OR standard `page.url` with `pushState`/`replaceState`.

| Context | API |
|---------|-----|
| Load Function (Server) | `url.searchParams.get()` |
| Components (Client) | `$page.url.searchParams` or `sveltekit-search-params` stores |

**Benefits:**
- Deep linking support
- Browser history integration
- Server-Side Rendering (SSR) compatibility

See [URL State](url_state.md) for detailed patterns.

---

### 2. Server State - Database Data

**Default Approach:** `load` functions in `+page.server.ts`.

Data flows one way: `Server (API)` -> `Load Function` -> `Page Prop (data)`.

**Revalidation:**
- `invalidateAll()`: Reloads all load functions.
- `invalidate('tag')`: Reloads specific dependencies.
- `applyAction`: Updates data after form submission automatically.

---

### 3. Local Feature State - Complex UI Logic

For complex UI state specific to one feature (e.g. multi-step form wizard, interactive dashboard).

**Use:** Svelte 5 Runes (`$state`, `$derived`, `.svelte.ts` modules).

```typescript
// features/chart/logic.svelte.ts
export class ChartState {
  hoverIndex = $state<number | null>(null);
  
  constructor(initialData: any[]) {
     // ...
  }
}
```

---

### 4. Global State - App-Wide Shared

Only for truly global, app-wide data (User Session, Theme, Toasts).

**Use:**
- **Shared Runes:** Create a global singleton instance in a `.svelte.ts` file.
- **Context API:** `setContext` / `getContext` (less common now with Runes but still useful for component sub-trees).

```typescript
// src/lib/stores/theme.svelte.ts
export const theme = $state({ current: 'light' });
```
