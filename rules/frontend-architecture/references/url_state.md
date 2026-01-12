# URL State Reference

Patterns for type-safe URL state management using `sveltekit-search-params` or `svelte-query-params`.

---

## Pattern: Typed Stores

Using libraries like `sveltekit-search-params` allows binding Svelte stores directly to URL parameters.

```typescript
// src/lib/features/search/params.ts
import { queryParam, ssp } from 'sveltekit-search-params';

export const searchParams = queryParam({
  q: ssp.string(''),
  page: ssp.number(1),
  sort: ssp.string('desc'),
});
```

### Usage in Component

```svelte
<!-- src/routes/search/+page.svelte -->
<script>
  import { searchParams } from '$lib/features/search/params';
  
  // Two-way binding automatically updates URL
  let { q, page, sort } = $searchParams;
</script>

<input bind:value={$searchParams.q} />
```

---

## Server-Side Access

In `+page.server.ts`, access via standard `url.searchParams`.

```typescript
// src/routes/search/+page.server.ts
export async function load({ url }) {
  const q = url.searchParams.get('q') || '';
  const page = Number(url.searchParams.get('page')) || 1;
  const sort = url.searchParams.get('sort') || 'desc';

  // Fetch data based on params
  return { 
    results: await searchApi.search(q, page, sort) 
  };
}
```

---

## Native Approach (Advanced)

If avoiding libraries, use `goto` with `replaceState` or `pushState`.

```typescript
import { goto } from '$app/navigation';
import { page } from '$app/stores';

function updateParam(key: string, value: string) {
  const url = new URL($page.url);
  url.searchParams.set(key, value);
  goto(url, { keepFocus: true, noScroll: true });
}
```
