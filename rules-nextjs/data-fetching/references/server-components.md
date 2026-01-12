# Server Components & Data Fetching

## Core Principle
Next.js 16+ uses React Server Components (RSC) by default. Data fetching should happen on the server to reduce client-side JavaScript and waterfall requests.

## Rule: NEVER Use Server Actions for Data Fetching
Server Actions (`'use server'`) are **only for mutations** (POST). Using them for GET requests breaks caching, streaming, and optimization.

### ❌ Anti-Pattern
```typescript
// actions.ts
'use server'
export async function getData() { ... } // DON'T DO THIS for reading data
```

### ✅ Correct Pattern
Use `server-only` async functions called directly from Server Components.
```typescript
import 'server-only'
export async function getData() { ... }
```

## Client-Side Fallbacks
Switch to Client Fetching only when:
1.  **Real-time updates** (polling/WebSockets)
2.  **Heavy interaction** (autocomplete, complex dashboards)
3.  **Browser-only APIs** (Geolocation, LocalStorage)

### Recommended Tools
-   **TanStack Query**: For complex client-side state, polling, and hydration.
-   **SWR**: For simpler client-side fetching.
