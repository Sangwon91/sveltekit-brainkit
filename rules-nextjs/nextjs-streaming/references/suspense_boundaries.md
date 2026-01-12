---
title: Suspense Boundary Placement
description: Strategies for placing Suspense boundaries to enable parallel streaming.
---

# Suspense Boundary Placement

Place Suspense boundaries **as close to the data source as possible**.

## Granular vs. Single Boundary

### ❌ Anti-Pattern: Single Large Boundary
Wraps the entire page or large section in one Suspense.

```typescript
// ❌ Sequential Loading (Waterfall potential)
<Suspense fallback={<PageSkeleton />}>
  <AllContent />
</Suspense>
```
*Result:* User waits for the *slowest* data source before seeing anything.

### ✅ Correct Pattern: Granular Boundaries
Wraps individual independent data-fetching components.

```typescript
// ✅ Parallel Loading
<Suspense fallback={<HeaderSkeleton />}>
  <Header />
</Suspense>
<Suspense fallback={<FeedSkeleton />}>
  <Feed />
</Suspense>
<Suspense fallback={<SidebarSkeleton />}>
  <Sidebar />
</Suspense>
```
*Result:* Fast components load immediately; slow ones stream in independently. Static shell is visible instantly.

## Fallback UI Design

1.  **Skeleton Matching:** Fallback should match the dimensions and layout of the loading content to prevent layout shift.
2.  **Meaningful Loading:** Use skeletons representing the UI structure (cards, lists) rather than generic spinners.

```typescript
// ✅ GOOD: Structure matching
function ChartSkeleton() {
  return <div className="h-[300px] w-full bg-muted rounded-lg animate-pulse" />
}

// ❌ BAD: Generic spinner (causes layout shift)
function ChartSkeleton() {
  return <Spinner />
}
```

## Visualization

**Parallel Streaming:**
```text
┌─────────────────────────────────────┐
│ <Suspense><UserProfile /></Suspense>│ 500ms (Done)
│ <Suspense><OrderList /></Suspense>  │ 800ms (Streaming...)
│ <Suspense><Analytics /></Suspense>  │ 1200ms (Streaming...)
└─────────────────────────────────────┘
Result: User profile visible at 500ms, doesn't wait for Analytics.
```
