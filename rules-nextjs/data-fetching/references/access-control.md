# Access Control & Security

## Rule: Always Use `server-only`
The `server-only` package prevents accidental exposure of sensitive server code to the client.

### How to use
1.  Install: `pnpm add server-only`
2.  Import at the top of **any file** touching DB, Secrets, or Internal APIs.

```typescript
// queries.ts
import 'server-only' // CRITICAL

const API_KEY = process.env.SECRET_KEY // Safe
```

## Security Design
-   **Hard Boundary**: If you import a `server-only` file into a Client Component (`'use client'`), the build will fail.
-   **Secrets**: Never expose API keys in `NEXT_PUBLIC_` variables unless truly public.
