# Directory Structure Reference

This document provides the complete directory structure for the Pragmatic Feature-based Architecture in Next.js 16+ (App Router).

---

## Complete File Tree

```
.
├── app/                      # [Shell] Routing & Page Composition only. Minimal logic.
│   ├── (auth)/login/page.tsx # Composition of Auth features
│   ├── dashboard/page.tsx    # Renders Feature Containers with Suspense
│   ├── api/                  # Route Handlers (only for external webhooks/integrations)
│   └── layout.tsx            # Global Root Layout
│
├── components/               # [Shared UI] Generic, domain-agnostic UI only.
│   ├── ui/                   # Design System (Shadcn UI: Button, Input, etc.)
│   ├── layout/               # Structural shells (Header, Sidebar - empty props)
│   └── error-boundary.tsx    # Global Error Handling
│
├── hooks/                    # [Shared Hooks] Domain-agnostic utilities (useDebounce, etc.)
│
├── lib/                      # [Infra & Config]
│   ├── supabase/             # DB Client setup
│   ├── db.ts                 # ORM Instance (Prisma/Drizzle)
│   ├── nuqs/                 # nuqs URL State Management
│   │   └── parsers.ts        # Common parsers (shared across features)
│   └── utils.ts              # Pure helper functions (cn, formatters)
│
├── stores/                   # [Global State] App-wide state only (User Session, Toasts).
│
└── features/                 # ★ [Core] The heart of the application.
    ├── {feature-name}/       # e.g., 'auth', 'stock-chart', 'trade'
    │   ├── components/       # UI Components (Server & Client)
    │   ├── hooks/            # Client-side logic (UI state, interactions)
    │   ├── services/         # Pure Business Logic & DAL (Server-Only)
    │   ├── parsers.ts        # nuqs parsers for URL state (type-safe)
    │   ├── queries.ts        # Data Fetching functions (server-only)
    │   ├── actions.ts        # Data Mutations - Server Actions
    │   ├── stores/           # Local Client State (Zustand) for this feature
    │   ├── types.ts          # Feature-specific type definitions
    │   ├── utils/            # Domain-specific math/helpers
    │   ├── index.ts          # ★ Public API (Client-safe exports)
    │   └── server.ts         # ★ Public API (Server-only exports)
```

---

## Directory Purposes

### `app/` - Routing Shell

- **Purpose:** Page composition and routing only
- **Rule:** Minimal logic. Import Feature Containers and wrap with `<Suspense>`
- **Pattern:** `page.tsx` is a thin orchestration layer

```typescript
// app/dashboard/page.tsx
import { Suspense } from "react";
import { DashboardContent } from "@/features/dashboard/server";

export default function DashboardPage({ searchParams }) {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <DashboardContent searchParams={searchParams} />
    </Suspense>
  );
}
```

### `components/` - Shared UI

- **Purpose:** Domain-agnostic, reusable UI components only
- **Contents:** Design system (Shadcn UI), layout shells, error boundaries
- **Promotion Rule:** Move feature-specific components here only when used by 3+ features

### `lib/` - Infrastructure

- **Purpose:** Configuration, clients, pure utilities
- **Contents:** Database clients, ORM instances, helper functions
- **Rule:** No React components, no business logic

### `features/` - Core Business Logic

- **Purpose:** The heart of the application
- **Each feature is a mini-library** with clear boundaries
- **See:** [Feature Anatomy](feature_anatomy.md) for detailed structure
