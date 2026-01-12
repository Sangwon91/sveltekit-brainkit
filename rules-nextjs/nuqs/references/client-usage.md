# Client-Side Usage (Read/Write)

Client Components handle user interactions that change the URL. `nuqs` provides hooks similar to `useState`.

## `useQueryState` (Single Param)

Use this for individual controls like a search bar or a single filter.

```typescript
'use client'
import { useQueryState } from 'nuqs'
import { parseAsString } from 'nuqs' // or import from your parsers file

export function SearchBar() {
  const [search, setSearch] = useQueryState('q', parseAsString.withDefault(''))

  return (
    <input 
      value={search}
      onChange={(e) => setSearch(e.target.value)}
    />
  )
}
```

## `useQueryStates` (Multiple Params)

Use this for forms, filter bars, or cohesive groups of parameters.

```typescript
'use client'
import { useQueryStates } from 'nuqs'
import { productParsers } from '@/features/products/parsers' // Shared parsers

export function ProductFilters() {
  const [filters, setFilters] = useQueryStates(productParsers)

  return (
    <>
      <select 
        value={filters.sort} 
        onChange={e => setFilters({ sort: e.target.value })}
      >
        <option value="newest">Newest</option>
        <option value="price_asc">Price Low-High</option>
      </select>
      
      <button onClick={() => setFilters({ page: 1, sort: 'newest' })}>
        Reset
      </button>
    </>
  )
}
```

## Setup Options

The `.withOptions` modifier or hook options object allows configuring behavior.

-   **history**: `'replace'` (default) or `'push'`.
-   **shallow**: `true` (default) or `false`.
-   **throttle**: Limit frequency of updates.

```typescript
const [value, setValue] = useQueryState('q', {
  history: 'push', // Create new history entry
  shallow: false   // Trigger server re-render
})
```

## Throttling and Debouncing

For high-frequency inputs (like text search or sliders), use the `throttle` or `debounce` helpers in the `limitUrlUpdates` option.

> **Note**: `throttleMs` is deprecated. Use `limitUrlUpdates`.

```typescript
import { throttle, debounce } from 'nuqs'

// Throttling (Sliders)
const [value, setValue] = useQueryState('slider', 
  parseAsInteger.withOptions({
    limitUrlUpdates: throttle(100) // Update max once per 100ms
  })
)

// Debouncing (Search inputs)
const [search, setSearch] = useQueryState('q', 
  parseAsString.withOptions({
    limitUrlUpdates: debounce(500) // Update 500ms AFTER user stops typing
  })
)
```
