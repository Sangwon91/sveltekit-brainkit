# URL State Reference

Patterns for type-safe URL state management using **native SvelteKit APIs**.

> [!NOTE]
> SvelteKit 5.1+ 부터 `$app/state`가 도입되었습니다. 기존 `$app/stores`도 호환되지만, Svelte 5 Runes와 함께 사용할 때는 `$app/state`가 더 자연스럽습니다.

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

## Client-Side Access (Svelte 5)

Use `page` from `$app/state` with `$derived` for reactive URL state.

```svelte
<!-- src/routes/search/+page.svelte -->
<script lang="ts">
  import { page } from '$app/state';
  import { goto } from '$app/navigation';
  
  let { data } = $props();
  
  // ✅ Svelte 5: $derived for reactive values
  const q = $derived(page.url.searchParams.get('q') || '');
  
  function updateSearch(newQuery: string) {
    const url = new URL(page.url);
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

/**
 * URL 파라미터를 패치하고 navigation을 수행합니다.
 * @param currentUrl - 현재 URL ($app/state의 page.url)
 * @param params - 설정할 파라미터 (null이면 삭제)
 */
export function patchUrlParams(
  currentUrl: URL,
  params: Record<string, string | null>
) {
  const url = new URL(currentUrl);
  
  Object.entries(params).forEach(([key, value]) => {
    if (value === null || value === '') {
      url.searchParams.delete(key);
    } else {
      url.searchParams.set(key, value);
    }
  });
  
  goto(url, { keepFocus: true, noScroll: true, replaceState: true });
}
```

Usage in component:

```svelte
<script lang="ts">
  import { page } from '$app/state';
  import { patchUrlParams } from '$lib/utils/url';
</script>

<button onclick={() => patchUrlParams(page.url, { page: '2' })}>
  Page 2
</button>
```

---

## Type-Safe Params (Advanced)

For type-safe URL params, create a typed helper using `$derived`.

```svelte
<!-- Usage in component -->
<script lang="ts">
  import { page } from '$app/state';

  interface SearchParams {
    q: string;
    page: number;
    sort: 'asc' | 'desc';
  }

  // ✅ Type-safe derived params
  const searchParams = $derived<SearchParams>({
    q: page.url.searchParams.get('q') || '',
    page: Number(page.url.searchParams.get('page')) || 1,
    sort: (page.url.searchParams.get('sort') as 'asc' | 'desc') || 'desc'
  });
</script>

<p>Searching for: {searchParams.q}</p>
```
