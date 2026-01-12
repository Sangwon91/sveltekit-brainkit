# Client-Side Integration (TanStack Query)

## Role: Server Cache vs Client Cache

Cache Components handle **Server-Level** caching. TanStack Query handles **Client-Level** (browser) caching.

- **Server (Cache Components)**: `'use cache'`, `cacheLife`. Focus on shared data, static shell, origin protection.
- **Client (TanStack Query)**: `staleTime`, `queryKey`. Focus on user interaction, instant feedback, optimistic updates.

Using both is **Layering**, not duplication.

## Architecture Principles

1.  **Server Cache protects Origin**: Reduces load on DB/API.
2.  **Client Cache improves UX**: Responses are instant for the user.
3.  **Unified Invalidation**: Major events should invalidate **BOTH** layers.

## Pattern: Server Cached Query utilized by Client

### 1. Server Side (`'use cache'`)

```typescript
// features/news/queries.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

export async function fetchLatestNews() {
  'use cache: remote'
  cacheLife('minutes')
  cacheTag('news')

  const res = await fetch(process.env.BACKEND_URL + '/news')
  return res.json()
}
```

### 2. API Route Wrapper

```typescript
// app/api/news/route.ts
import { NextResponse } from 'next/server'
import { fetchLatestNews } from '@/features/news/queries'

export async function GET() {
  // Uses the server cache
  const data = await fetchLatestNews()
  return NextResponse.json(data)
}
```

### 3. Client Side (TanStack Query)

```typescript
// features/news/client/queries.ts
import { queryOptions } from '@tanstack/react-query'
import apiClient from '@/lib/api/client'

export function newsQueryOptions() {
  return queryOptions({
    queryKey: ['news'],
    queryFn: () => apiClient.get('/news').then(r => r.data),
    staleTime: 2 * 60 * 1000, 
  })
}
```

## Invalidation Flow

When data changes (e.g., publishing news), you must invalidate BOTH caches to ensure consistency.

### 1. Server Action (Invalidates Server Cache)

```typescript
// features/news/actions.ts
'use server'
import { revalidateTag } from 'next/cache'

export async function publishNews(id: string) {
  await db.news.update({ where: { id }, data: { published: true } })
  // Invalidate Server Cache
  revalidateTag('news') 
}
```

### 2. Client Component (Invalidates Client Cache)

```typescript
// features/news/client/PublishButton.tsx
'use client'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { publishNewsAction } from '../actions' // or via API

export function PublishButton({ id }) {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: () => publishNewsAction(id),
    onSuccess: async () => {
      // Invalidate Browser Cache
      await queryClient.invalidateQueries({ queryKey: ['news'] })
    }
  })
  
  return <button onClick={() => mutation.mutate()}>Publish</button>
}
```
