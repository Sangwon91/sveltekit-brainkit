# Type-Safe Parsers

Defining parsers is the foundation of `nuqs`. They ensure that URL parameters are correctly converted to TypeScript types and validated.

## Defining Parsers

Parsers should be defined in a dedicated `parsers.ts` file (either per-feature or global).

```typescript
// features/products/parsers.ts
import { 
  parseAsInteger, 
  parseAsString, 
  parseAsStringEnum, 
  parseAsBoolean,
  createSearchParamsCache 
} from 'nuqs/server'

export const productParsers = {
  // String with default
  search: parseAsString.withDefault(''),
  
  // Integer with default
  page: parseAsInteger.withDefault(1),
  
  // Enum (Union Type)
  sort: parseAsStringEnum(['asc', 'desc', 'newest']).withDefault('newest'),
  
  // Boolean
  inStock: parseAsBoolean.withDefault(false),
} as const

// REQUIRED: Create and export cache for Server Component usage
export const productCache = createSearchParamsCache(productParsers)
```

## Common Parser Types

| Parser | TypeScript Type | Use Case |
|--------|-----------------|----------|
| `parseAsString` | `string` | Search queries, IDs |
| `parseAsInteger` | `number` | Pagination, Limits |
| `parseAsFloat` | `number` | Price ranges, coordinates |
| `parseAsBoolean` | `boolean` | Toggles, flags |
| `parseAsStringEnum` | `T` (Union) | Sorting, filtering categories |
| `parseAsArrayOf` | `T[]` | Multi-select filters |
| `parseAsIsoDateTime`| `Date` | Date pickers |

## Custom Parsers

For complex types (like objects or special formats), use `createParser`.

```typescript
import { createParser } from 'nuqs/server'

// Parser for "2024-01-01..2024-01-31" style ranges
export const dateRangeParser = createParser({
  parse: (value: string) => {
    const [start, end] = value.split('..')
    if (!start || !end) return null
    return { start: new Date(start), end: new Date(end) }
  },
  serialize: (value: { start: Date; end: Date }) => {
    return `${value.start.toISOString()}..${value.end.toISOString()}`
  },
}).withDefault({ start: new Date(), end: new Date() })
```

## Best Practices

1.  **Centralize Definitions**: Keep parsers in `parsers.ts` files.
2.  **Export for Client**: Export the parsers object so Client Components can import it (ensures server/client align).
3.  **Always set Defaults**: `.withDefault()` prevents `null` checks everywhere in your code.
