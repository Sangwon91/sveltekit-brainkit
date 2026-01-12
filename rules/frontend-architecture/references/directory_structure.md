# Directory Structure Reference

This document provides the complete directory structure for the Pragmatic Feature-based Architecture in SvelteKit v5.

---

## Complete File Tree

```
src/
├── routes/                   # [Shell] Routing & Page Composition only.
│   ├── (auth)/login/
│   │   ├── +page.svelte      # Page View
│   │   └── +page.server.ts   # Form Actions & Data Loading
│   ├── dashboard/
│   │   ├── +page.svelte      # Renders Feature Components
│   │   └── +page.server.ts   # Data Loading for Dashboard
│   ├── +layout.svelte        # Global Root Layout
│   └── +error.svelte         # Global Error Handling
│
├── lib/                      # [Shared Code]
│   ├── server/               # [Server Infra] DB, Secrets (Server-only safe)
│   │   ├── db.ts             # ORM Instance
│   │   └── secrets.ts
│   │
│   ├── components/           # [Shared UI] Domain-agnostic UI (e.g. shadcn-svelte)
│   │   ├── ui/
│   │   └── layout/
│   │
│   ├── utils/                # [Shared Utilities] Pure helpers
│   │
│   └── features/             # ★ [Core] The heart of the application.
│       ├── {feature-name}/   # e.g., 'auth', 'analytics'
│       │   ├── components/   # UI Components
│       │   ├── logic.svelte.ts # Client Logic & State (Runes)
│       │   ├── api.server.ts # Business Logic & DAL (Server-Only)
│       │   ├── types.ts      # Feature-specific types
│       │   ├── index.ts      # ★ Client-safe exports
│       │   └── server.ts     # ★ Server-only exports
│
└── app.d.ts                  # Global type definitions
```

---

## Directory Purposes

### `src/routes/` - Routing Shell

- **Purpose:** Page composition, routing, and data orchestration.
- **Rule:** Keep `+page.svelte` files thin. Delegate logic to Feature Modules.
- **Pattern:** `+page.server.ts` handles data loading and passes it to `+page.svelte`.

```typescript
// src/routes/dashboard/+page.server.ts
import { dashboardApi } from '$lib/features/dashboard/server';

export async function load({ locals }) {
  const data = await dashboardApi.getOverview(locals.user.id);
  return { data };
}
```

### `src/lib/components/` - Shared UI

- **Purpose:** Domain-agnostic, reusable UI components.
- **Contents:** Design system (shadcn-svelte), primitives.
- **Rule:** Do not put business logic here.

### `src/lib/server/` - Server Infrastructure

- **Purpose:** Database connections, sensitive configuration.
- **Protection:** SvelteKit prevents importing `$lib/server` modules into client-side code ($lib/server refers to the actual directory convention).

### `src/lib/features/` - Core Business Logic

- **Purpose:** Encapsulated feature modules.
- **See:** [Feature Anatomy](feature_anatomy.md) for detailed structure.
