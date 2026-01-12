---
alwaysApply: true
---

# Data Fetching Guidelines

## Philosophy
**Server-First, Cache-Explicit.**
Fetch data on the server close to the source to minimize waterfalls and bundle size. Use Explicit Caching (`use cache`) to enable Partial Prerendering (PPR) and maximize the Static Shell.

## Core Knowledge (the "Cheat Sheet")

### ✅ Do's
-   **Fetch in Server Components**: Use async/await directly in Server Components.
-   **Use `server-only`**: Import strictly in data access layers.
-   **Use `use cache`**: Explicitly declare caching for async functions.
-   **Use `use cache: remote`**: For **Serverless/Vercel** deployments to ensure persistence.
-   **Use `<Suspense>`**: For streaming slow or personalized data.
-   **Use `nuqs`**: For type-safe URL state management.

### ❌ Don'ts
-   **NEVER use Server Actions for GET**: They are for mutations (POST) only.
-   **Don't fetch in Client Components** unless necessary (real-time, heavy interaction).
-   **Don't leak secrets**: Never pass full DB objects to the client; serializable DTOs only.

## Tech Stack
-   **Next.js 16+** (App Router)
-   **React Server Components (RSC)**
-   **server-only**
-   **nuqs** (URL State)

## Workflow
1.  **Define Parsers**: Create `parsers.ts` with *nuqs* for URL params.
2.  **Create Query**: Write async function in `queries.ts`.
    -   Add `import 'server-only'`.
    -   Add `'use cache'` (or `'use cache: remote'`).
    -   Define `cacheLife` and `cacheTag`.
3.  **Fetch**: Call query in Server Component (Page or Container).
4.  **Suspend**: Wrap component in `<Suspense>` if it blocks the shell.
5.  **Render**: Pass serialized data to Client Components if interactivity is needed.

## References
-   See [Server Component Patterns](references/server-components.md)
-   See [Cache Components Strategy](references/cache-components.md)
-   See [Access Control](references/access-control.md)
-   See [Serialization Boundaries](references/serialization.md)
-   See [URL State Integration](references/url-state.md)
