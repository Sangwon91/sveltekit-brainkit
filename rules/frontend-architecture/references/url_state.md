# URL State Reference

Patterns for type-safe URL state management using **native SvelteKit APIs**.

---

## Server-Side Access

In `+page.server.ts`, access via standard `url.searchParams`.

```typescript
// src/routes/search/+page.server.ts
import { searchItems } from '$lib/features/search/api.server';

export async function load({ url }) {
  const q = url.searchParams.get('q') || '';
  const page = Number(url.searchParams.get('page')) || 1;
  const sort = url.searchParams.get('sort') || 'desc';

  return { 
    results: await searchItems(q, page, sort),
    params: { q, page, sort }
  };
}
```

---

## Client-Side Access

Use `$page` store from `$app/stores`.

```svelte
<!-- src/routes/search/+page.svelte -->
<script lang="ts">
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  
  let { data } = $props();
  
  // Read from URL
  $: q = $page.url.searchParams.get('q') || '';
  
  function updateSearch(newQuery: string) {
    const url = new URL($page.url);
    url.searchParams.set('q', newQuery);
    goto(url, { keepFocus: true, noScroll: true });
  }
</script>

<input 
  value={q} 
  oninput={(e) => updateSearch(e.currentTarget.value)} 
/>
```

---

## Reusable URL State Helper

Create a helper function for common URL manipulation patterns.

```typescript
// src/lib/utils/url.ts
import { goto } from '$app/navigation';
import { page } from '$app/stores';
import { get } from 'svelte/store';

export function updateSearchParam(key: string, value: string | null) {
  const url = new URL(get(page).url);
  
  if (value === null || value === '') {
    url.searchParams.delete(key);
  } else {
    url.searchParams.set(key, value);
  }
  
  goto(url, { keepFocus: true, noScroll: true, replaceState: true });
}
```

Usage:
```svelte
<script>
  import { updateSearchParam } from '$lib/utils/url';
</script>

<button onclick={() => updateSearchParam('page', '2')}>
  Page 2
</button>
```

---

## Type-Safe Params (Advanced)

For type-safe URL params, create a typed helper.

```typescript
// src/lib/features/search/params.ts
import { page } from '$app/stores';
import { derived } from 'svelte/store';

export interface SearchParams {
  q: string;
  page: number;
  sort: 'asc' | 'desc';
}

export const searchParams = derived(page, ($page) => ({
  q: $page.url.searchParams.get('q') || '',
  page: Number($page.url.searchParams.get('page')) || 1,
  sort: ($page.url.searchParams.get('sort') as 'asc' | 'desc') || 'desc'
}));
```
