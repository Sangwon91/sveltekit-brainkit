# Next.js 16+ Integration

## Core Rule: Client Components Only

D3 relies entirely on Browser APIs (`document`, `window`, `CanvasRenderingContext2D`). It **cannot** run on the server.

-   **Feature Components**: Always mark D3 chart files with `'use client'`.
-   **Data Fetching**: Keep data fetching in **Server Components** and pass data down as props.

## Architecture Pattern

```
features/
└── crypto-chart/
    ├── components/
    │   ├── ChartContainer.tsx      # Server Component (Fetches Data)
    │   ├── PriceChart.tsx          # Client Component (D3 Logic)
    │   └── Controls.tsx            # Client Component (UI)
    ├── queries.ts                  # Server-only data fetching
    └── index.ts                    # Public Export
```

### Server Component (Data Fetching)

```typescript
// features/crypto-chart/components/ChartContainer.tsx
import { fetchCryptoHistory } from '../queries';
import { PriceChart } from './PriceChart';

export async function ChartContainer({ symbol }) {
  // ✅ Fetch on server, close to DB
  const data = await fetchCryptoHistory(symbol);
  
  return (
    <div className="h-[400px]">
       <PriceChart data={data} />
    </div>
  );
}
```

### Client Component (Visualization)

```typescript
// features/crypto-chart/components/PriceChart.tsx
'use client'; // 👈 Essential

import { useLayoutEffect, useRef } from 'react';
import * as d3 from 'd3';

export function PriceChart({ data }) {
  const ref = useRef(null);
  // ... d3 logic ...
  return <svg ref={ref} />;
}
```

## Suspense Integration

Because D3 charts often depend on significant data, wrap the Server Component in `Suspense` at the Page level or Feature level.

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { ChartContainer } from '@/features/crypto-chart/server';
import { SkeletonChart } from '@/components/skeletons';

export default function Page() {
  return (
    <Suspense fallback={<SkeletonChart />}>
      <ChartContainer symbol="BTC" />
    </Suspense>
  );
}
```

## Dynamic Imports (Optional)

If the D3 library itself is huge or the chart is below the fold, you can lazy load the Client Component.

```typescript
import dynamic from 'next/dynamic';

const PriceChart = dynamic(
  () => import('./PriceChart').then(mod => mod.PriceChart),
  { ssr: false }
);
```
