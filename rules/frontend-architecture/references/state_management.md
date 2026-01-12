# State Management Reference

Follow this priority order for managing state in SvelteKit v5.

---

## State Hierarchy (Priority Order)

### 1. URL (Query Params) - Shareable State

For filters, pagination, tabs, and any state that should be bookmarkable/shareable.

**Use:** Native SvelteKit APIs (`$page.url`, `goto`).

| Context | API |
|---------|-----|
| Load Function (Server) | `url.searchParams.get()` |
| Components (Client) | `$page.url.searchParams` |

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

**Use:** Svelte 5 Runes (`$state`, `$derived`) in `.svelte.ts` modules with **함수형 패턴**.

```typescript
// features/chart/state.svelte.ts
export function createChartState(initialData: any[]) {
  let hoverIndex = $state<number | null>(null);
  let data = $state(initialData);
  
  const activeItem = $derived(
    hoverIndex !== null ? data[hoverIndex] : null
  );
  
  function setHover(index: number | null) {
    hoverIndex = index;
  }
  
  return {
    get hoverIndex() { return hoverIndex; },
    get activeItem() { return activeItem; },
    setHover
  };
}
```

---

### 4. Global State - App-Wide Shared

Only for truly global, app-wide data (User Session, Theme, Toasts).

**Use:**
- **Shared Runes:** Create a global function in a `.svelte.ts` file.
- **Context API:** `setContext` / `getContext` for component sub-trees.

```typescript
// src/lib/stores/theme.svelte.ts
function createThemeState() {
  let current = $state<'light' | 'dark'>('light');
  
  function toggle() {
    current = current === 'light' ? 'dark' : 'light';
  }
  
  return {
    get current() { return current; },
    toggle
  };
}

// Singleton for global use
export const theme = createThemeState();
```
