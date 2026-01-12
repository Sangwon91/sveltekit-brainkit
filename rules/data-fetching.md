---
trigger: always_on
glob: "*"
description: "Data Fetching Guidelines for SvelteKit v5"
---

# Data Fetching Guidelines

## Philosophy
**Server-First, Native-Preferred.**
Fetch data on the server via `load` functions. Let SvelteKit handle SSR, streaming, and caching. Use client-side fetching only when the server cannot satisfy the requirement.

## Core Knowledge (Cheat Sheet)

### ✅ Do's
- **Use `load` functions**: `+page.server.ts` for sensitive data, `+page.ts` for client-shareable.
- **Use streaming**: Return unwrapped promises for non-critical data (see below).
- **Use Form Actions**: For mutations (POST, DELETE, PUT).
- **Use native URL state**: `$page.url.searchParams` + `goto` for filters/pagination.
- **Use Zod**: Validate `params` and `url.searchParams` in load functions.

### ❌ Don'ts
- **NEVER `fetch` from client components** if `load` can provide the data.
- **Don't bypass Form Actions**: Use `fetch('/api')` only for non-mutation XHR needs.
- **Don't leak secrets**: Use `+page.server.ts` for DB calls and API keys.

## Tech Stack
- **SvelteKit v5** (load functions, Form Actions)
- **Svelte 5 Runes** ($state, $derived for local state)
- **Zod** (validation in load)

> [!WARNING]
> **TanStack Query**: 정말 필요한 경우(polling, infinite scroll, cross-route cache)에만 도입하세요.
> SvelteKit의 `load` + `invalidate` 시스템과 중복되는 캐싱 레이어를 추가하므로, 대부분의 경우 네이티브 패턴으로 충분합니다.

---

## Server Data Fetching

### Standard Pattern: `+page.server.ts`

All sensitive data fetching should occur in `+page.server.ts`. Server load functions provide a clear boundary for sensitive operations.

```typescript
// src/routes/products/+page.server.ts
import { z } from 'zod';
import { error } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

const searchSchema = z.object({
  page: z.coerce.number().min(1).default(1),
  category: z.string().optional()
});

export const load: PageServerLoad = async ({ url, locals }) => {
  const params = searchSchema.parse(Object.fromEntries(url.searchParams));
  
  // Server-only: safe to use secrets
  const products = await db.product.findMany({
    where: { category: params.category },
    skip: (params.page - 1) * 20,
    take: 20
  });
  
  return { products, page: params.page };
};
```

### Why `+page.server.ts` over `+page.ts`?

| Feature | `+page.server.ts` | `+page.ts` |
|---------|-------------------|------------|
| Runs on | Server only | Server (SSR) + Client (CSR) |
| Access secrets | ✅ Yes | ❌ No (leaked to client) |
| Direct DB access | ✅ Yes | ❌ No |
| Bundle size | ✅ Smaller | Larger (code shipped) |
| Client-side rerun | ❌ No | ✅ Yes (on navigation) |

**Rule**: Default to `+page.server.ts`. Use `+page.ts` only for data that must also be fetched client-side during navigation.

---

## Streaming (Non-Blocking Loads)

SvelteKit streams non-critical data by returning **unwrapped promises** from load functions.

```typescript
// src/routes/dashboard/+page.server.ts
export const load: PageServerLoad = async ({ locals }) => {
  return {
    // Critical: blocks rendering (awaited)
    user: await db.getUser(locals.user.id),
    
    // Non-Critical: streamed to client (NOT awaited)
    analytics: db.getAnalytics(locals.user.id),
    recommendations: db.getRecommendations(locals.user.id)
  };
};
```

### Handling in Component

```svelte
<script lang="ts">
  let { data } = $props();
</script>

<h1>Welcome, {data.user.name}</h1>

{#await data.analytics}
  <p>Loading analytics...</p>
{:then results}
  <AnalyticsDashboard {results} />
{:catch error}
  <p>Failed to load analytics</p>
{/await}
```

---

## URL State Management

Use **native SvelteKit APIs**. No external libraries needed.

### Server-Side (Load Function)

```typescript
// src/routes/search/+page.server.ts
export const load: PageServerLoad = async ({ url }) => {
  const q = url.searchParams.get('q') || '';
  const page = Number(url.searchParams.get('page')) || 1;
  
  return {
    results: await searchService.search(q, page),
    query: q,
    page
  };
};
```

### Client-Side (Component)

```svelte
<script lang="ts">
  import { page } from '$app/state';
  import { goto } from '$app/navigation';
  
  let { data } = $props();
  
  // Svelte 5: $derived for reactive values from $app/state
  const q = $derived(page.url.searchParams.get('q') || '');
  
  function updateSearch(newQuery: string) {
    const url = new URL(page.url);
    url.searchParams.set('q', newQuery);
    goto(url, { replaceState: true, keepFocus: true, noScroll: true });
  }
</script>

<input value={q} oninput={(e) => updateSearch(e.currentTarget.value)} />
```

### Reusable Helper

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
    if (value === null) {
      url.searchParams.delete(key);
    } else {
      url.searchParams.set(key, value);
    }
  });
  
  goto(url, { replaceState: true, keepFocus: true, noScroll: true });
}

// Usage in component:
// import { page } from '$app/state';
// patchUrlParams(page.url, { q: 'new query' });
```

---

## Mutations (Form Actions)

**Always use Form Actions for mutations.** They handle:
- Progressive enhancement (works without JS)
- CSRF protection
- Automatic `load` revalidation

```typescript
// src/routes/posts/[id]/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';

export const actions: Actions = {
  update: async ({ request, params, locals }) => {
    const formData = await request.formData();
    const title = formData.get('title');
    
    if (!title) {
      return fail(400, { error: 'Title required' });
    }
    
    await db.post.update({
      where: { id: params.id },
      data: { title: String(title) }
    });
    
    return { success: true };
  },
  
  delete: async ({ params }) => {
    await db.post.delete({ where: { id: params.id } });
    redirect(303, '/posts');
  }
};
```

### Usage in Component

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  let { data, form } = $props();
</script>

<form method="POST" action="?/update" use:enhance>
  <input name="title" value={data.post.title} />
  {#if form?.error}<p class="error">{form.error}</p>{/if}
  <button type="submit">Save</button>
</form>
```

---

## Client-Side Fetching (When Needed)

Only use client-side fetching for:
1. **Real-time updates** (polling, WebSockets)
2. **Infinite scroll / "Load More"**
3. **Browser-only APIs** (Geolocation, etc.)
4. **Complex client-side interactions** (search-as-you-type with debounce)

### Option 1: Native `fetch` with Runes

For simple cases, use `$state` and `$effect` with **debouncing**.

```svelte
<script lang="ts">
  let searchQuery = $state('');
  let results = $state<SearchResult[]>([]);
  let loading = $state(false);
  
  // Debounced search to avoid excessive requests
  let debounceTimer: ReturnType<typeof setTimeout>;
  
  $effect(() => {
    if (!searchQuery) {
      results = [];
      return;
    }
    
    // Clear previous timer
    clearTimeout(debounceTimer);
    
    const controller = new AbortController();
    
    debounceTimer = setTimeout(() => {
      loading = true;
      fetch(`/api/search?q=${encodeURIComponent(searchQuery)}`, {
        signal: controller.signal
      })
        .then(r => r.json())
        .then(data => { results = data; })
        .finally(() => { loading = false; });
    }, 300); // 300ms debounce
    
    return () => {
      clearTimeout(debounceTimer);
      controller.abort();
    };
  });
</script>
```

> [!TIP]
> debounce 로직이 반복된다면 `$lib/utils/debounce.ts`로 추출하세요.

### Option 2: TanStack Query (Complex Cases)

For advanced needs (polling, infinite scroll, cross-route cache). **대부분의 경우 Option 1로 충분합니다.**

```svelte
<!-- src/routes/search/+page.svelte -->
<script lang="ts">
  import { createQuery } from '@tanstack/svelte-query';
  
  let searchQuery = $state('');
  
  // Reactive query using getter functions
  const searchResults = createQuery(() => ({
    queryKey: ['search', searchQuery],
    queryFn: async () => {
      const res = await fetch(`/api/search?q=${searchQuery}`);
      return res.json();
    },
    staleTime: 60 * 1000,
    enabled: searchQuery.length > 0
  }));
</script>

{#if searchResults.isPending}
  <p>Loading...</p>
{:else if searchResults.data}
  <!-- render results -->
{/if}
```

> [!IMPORTANT]
> TanStack Query v5는 Svelte 5 Runes와 함께 사용 시 `createQuery(() => options)` 패턴을 사용합니다.
> 반응성을 위해 options 객체를 getter 함수로 감싸세요.

---

## Serialization & Security

### Data Boundary

All data returned from `load` functions is automatically serialized. Keep it thin.

```typescript
// ✅ GOOD: Return only needed fields
return {
  user: {
    id: user.id,
    name: user.name,
    email: user.email
  }
};

// ❌ BAD: Full ORM object with methods/relations
return { user }; // May leak internals
```

### Server-Only Protection

SvelteKit naturally protects `+page.server.ts` code. For shared utilities that must never run on client:

```typescript
// src/lib/server/db.ts
// File path includes 'server' - SvelteKit prevents client import

import { DATABASE_URL } from '$env/static/private';
// $env/static/private is automatically server-only
```

---

## Decision Tree

```
Need data for page render?
├── Yes → Use load function (+page.server.ts)
│   ├── Sensitive (DB/secrets)? → +page.server.ts ✅
│   └── Safe for client? → +page.ts (runs both)
│
└── No (client-only interaction)
    ├── Simple fetch? → $effect + fetch
    └── Complex (polling, cache)? → TanStack Query
```
