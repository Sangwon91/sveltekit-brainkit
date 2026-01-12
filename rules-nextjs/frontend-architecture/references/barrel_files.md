# Barrel Files Reference

Features expose **two entry points** to handle the server/client boundary safely.

---

## `index.ts` - Client-Safe Exports

Exports modules that can be safely imported from **any context** (Client Components, Server Components, page.tsx):

```typescript
// features/dashboard/index.ts
/** Dashboard Feature - Client-safe API */

// Client Components (View/Presentational)
export { DashboardView } from "./components/DashboardView";
export { DashboardCard } from "./components/DashboardCard";

// Hooks (client-side logic)
export { useDashboardFilters } from "./hooks/useDashboardFilters";

// Parsers (safe for both server and client)
export { momentumParsers, momentumCache } from "./parsers";

// Types (always safe)
export type { DashboardData, DashboardFilter } from "./types";

// NOTE: Server-only exports (DashboardContent, queries) are in server.ts
```

---

## `server.ts` - Server-Only Exports

Exports modules that depend on `server-only`. Importing from Client Components causes **build errors** (intentional protection):

```typescript
// features/dashboard/server.ts
/** Dashboard Feature - Server-only API */

// Server Component Containers
export { DashboardContent } from "./components/DashboardContent";

// Query Functions
export { fetchDashboardData } from "./queries";

// Services (if needed externally)
export { calculateMetrics } from "./services/metrics-service";
```

---

## Why Two Files?

The `server-only` package creates a **compile-time boundary**:

```
┌─────────────────────────────────────────────────────────────────┐
│  index.ts exports                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ DashboardView   │  │ useDashboard    │  │ Types           │ │
│  │ (use client)    │  │ Filters         │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  Can be imported from: Client Components, Server Components,    │
│  page.tsx, anywhere                                             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  server.ts exports                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ DashboardContent│  │ fetchDashboard  │  │ Services        │ │
│  │ (async, fetches)│  │ Data (queries)  │  │ (server-only)   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                    ↓                                            │
│              imports queries.ts                                 │
│                    ↓                                            │
│              import "server-only"                               │
│                                                                 │
│  Can ONLY be imported from: Server Components, page.tsx         │
│  ❌ Importing from Client Component = BUILD ERROR               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Import Patterns Quick Reference

```typescript
// ✅ From Client Components
import { DashboardView } from "@/features/dashboard";
import { useDashboardFilters } from "@/features/dashboard";

// ✅ From page.tsx (Server Component)
import { DashboardContent } from "@/features/dashboard/server";
import { fetchDashboardData } from "@/features/dashboard/server";

// ✅ Parsers (can be imported from both server and client)
import { momentumParsers } from "@/features/momentum-analysis";

// ❌ WRONG: Will cause build error
// In a Client Component:
import { DashboardContent } from "@/features/dashboard/server"; // BUILD ERROR!
```
