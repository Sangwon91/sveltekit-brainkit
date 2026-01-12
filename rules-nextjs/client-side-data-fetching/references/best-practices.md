# Client-Side Data Fetching: Best Practices

## 1. Query Key Strategy

Use **hierarchical** keys. This allows you to invalidate broad categories (e.g., all 'posts') or specific items (e.g., 'posts' for user '123').

```typescript
// ✅ GOOD: Hierarchical
['todos', { status: 'done', page: 1 }]
['posts', 'detail', id]
['user', userId, 'profile']

// ❌ BAD: Flat
['todosDonePage1']
['postDetail']
```

## 2. Stale Time & Caching

Configure `staleTime` based on data volatility.

- **0ms** (Default): Data is always stale. Refetches on every mount/window focus.
- **30s - 1m**: Dashboard metrics, lists. Good balance.
- **5m+**: User profile, settings, static config.
- **Infinity**: Data that never changes during session.

```typescript
// Global default (in QueryClient)
staleTime: 60 * 1000

// Per-query override
useQuery({
  queryKey: ['stock-price'],
  queryFn: fetchPrice,
  staleTime: 0, // Always fresh
  refetchInterval: 5000, // Poll every 5s
})
```

## 3. Performance & UX

### Prefetching
Preload data when the user hovers over a link or button.

```typescript
const queryClient = useQueryClient()

const onHover = () => {
  queryClient.prefetchQuery({
    queryKey: ['posts', nextId],
    queryFn: () => fetchPost(nextId),
  })
}
```

### Loading States
Avoid "Loading..." flickers.
- Use `isFetching` for background updates.
- Keep previous data while fetching new data (pagination).

```typescript
// Keep previous data during page change
useQuery({
  queryKey: ['projects', page],
  queryFn: fetchProjects,
  placeholderData: keepPreviousData,
})
```

### Type-Safe Error Handling
Don't rely on `any`.

```typescript
if (error) {
  // Axios error type narrowing
  if (axios.isAxiosError(error) && error.response?.status === 404) {
    return <NotFound />
  }
  return <GenericError />
}
```
