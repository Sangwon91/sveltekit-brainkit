# Client-Side Data Fetching: Advanced Patterns

## 1. Dependent Queries

Run queries sequentially when one depends on the other's data.

```typescript
const { data: user } = useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
})

const { data: posts } = useQuery({
  queryKey: ['posts', user?.id],
  queryFn: () => fetchPosts(user!.id),
  enabled: !!user, // Wait until user exists
})
```

## 2. Optimistic Updates

Update the UI immediately before the server responds.

```typescript
const mutation = useMutation({
  mutationFn: updatePost,
  onMutate: async (newPost) => {
    // 1. Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['posts'] })

    // 2. Snapshot previous value
    const previous = queryClient.getQueryData(['posts'])

    // 3. Optimistically update
    queryClient.setQueryData(['posts'], (old: Post[]) => [
      ...old,
      { ...newPost, id: 'temp-id' }
    ])

    return { previous }
  },
  onError: (err, newPost, context) => {
    // 4. Rollback on error
    queryClient.setQueryData(['posts'], context?.previous)
  },
  onSettled: () => {
    // 5. Always refetch to ensure consistency
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  }
})
```

## 3. Infinite Queries (Pagination)

Best for "Load More" or Infinite Scroll interfaces.

```typescript
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['posts', 'infinite'],
  queryFn: ({ pageParam = 1 }) => fetchPosts({ page: pageParam }),
  getNextPageParam: (lastPage, allPages) => {
    return lastPage.hasNextPage ? allPages.length + 1 : undefined
  },
  initialPageParam: 1,
})
```
