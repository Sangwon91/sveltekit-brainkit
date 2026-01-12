# Setup & Configuration

## Installation

```bash
pnpm add nuqs
```

## Root Layout Adapter

To enable `nuqs` functionality throughout your Next.js App Router application, you must wrap your application in the `NuqsAdapter`.

```typescript
// app/layout.tsx
import { NuqsAdapter } from 'nuqs/adapters/next/app'

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>
        <NuqsAdapter>
          {children}
        </NuqsAdapter>
      </body>
    </html>
  )
}
```

## Core Concept
`nuqs` provides a bridge between React state and the URL query string.
- **Server**: It parses search params into typed objects.
- **Client**: It provides hooks (`useQueryState`) that behave like `useState` but stay in sync with the URL.
