# Migration to Next.js 16 Cache Components

## Migration Table

| Legacy (Next.js 15) | Next.js 16+ Replacement |
|---------------------|------------------------|
| `export const dynamic = 'force-dynamic'` | Remove (default behavior) |
| `export const dynamic = 'force-static'` | `'use cache'` + `cacheLife('max')` |
| `export const revalidate = 60` | `cacheLife('minutes')` or custom config |
| `export const fetchCache = 'force-cache'` | Remove (use `'use cache'` scope) |
| `unstable_cache()` | `'use cache'` directive |
| React `cache()` for memoization | Request memoization is automatic for `fetch` |

## Before/After Examples

### Force Dynamic

**Before (Next.js 15)**
```typescript
// ❌ Next.js 15 - No longer needed
export const dynamic = 'force-dynamic'

export default async function Page() {
  const data = await fetchData()
  return <div>{data}</div>
}
```

**After (Next.js 16)**
```typescript
// ✅ Next.js 16 - Just remove it, pages are dynamic by default
export default async function Page() {
  const data = await fetchData()
  return <div>{data}</div>
}
```

### ISR with Revalidate

**Before (Next.js 15)**
```typescript
// ❌ Next.js 15
export const revalidate = 3600

export default async function Page() {
  const posts = await db.post.findMany()
  return <BlogList posts={posts} />
}
```

**After (Next.js 16)**
```typescript
// ✅ Next.js 16
import { cacheLife } from 'next/cache'

export default async function Page() {
  'use cache'
  cacheLife('hours')
  
  const posts = await db.post.findMany()
  return <BlogList posts={posts} />
}
```

### unstable_cache

**Before (Next.js 15)**
```typescript
// ❌ Next.js 15
import { unstable_cache } from 'next/cache'

const getCachedPosts = unstable_cache(
  async () => db.post.findMany(),
  ['posts'],
  { revalidate: 3600, tags: ['posts'] }
)
```

**After (Next.js 16)**
```typescript
// ✅ Next.js 16
import { cacheLife, cacheTag } from 'next/cache'

async function getCachedPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')
  
  return db.post.findMany()
}
```
