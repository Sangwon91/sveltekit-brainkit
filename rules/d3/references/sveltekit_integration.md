# SvelteKit Integration

## Core Rule: Browser-Only Execution

D3 relies entirely on Browser APIs (`document`, `window`, `CanvasRenderingContext2D`). It **cannot** run on the server.

-   **Chart Components**: Use `$effect` with element ref guards for all D3 logic. **No `onMount` or `browser` checks needed** in Svelte 5.
-   **Data Fetching**: Keep data fetching in **`+page.server.ts`** or **`+layout.server.ts`** and pass data via the `data` prop.

## Architecture Pattern

```
src/lib/features/
└── crypto-chart/
    ├── components/
    │   ├── ChartContainer.svelte   # Layout, handles loading states
    │   ├── PriceChart.svelte       # D3 visualization (browser-only)
    │   └── Controls.svelte         # UI controls
    ├── queries.ts                  # Data fetching utilities
    └── index.ts                    # Public exports
```

### Server Data Loading (`+page.server.ts`)

```typescript
// src/routes/dashboard/+page.server.ts
import { fetchCryptoHistory } from '$lib/features/crypto-chart/queries';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params }) => {
  // ✅ Fetch on server, close to DB
  const chartData = await fetchCryptoHistory('BTC');

  return {
    chartData
  };
};
```

### Page Component (`+page.svelte`)

```svelte
<!-- src/routes/dashboard/+page.svelte -->
<script lang="ts">
  import { PriceChart } from '$lib/features/crypto-chart';
  import type { PageData } from './$types';

  let { data }: { data: PageData } = $props();
</script>

<div class="h-[400px]">
  <PriceChart data={data.chartData} />
</div>
```

### Visualization Component (Browser-Only)

In Svelte 5, `$effect` only runs on the client, so you don't need `onMount` or `browser` checks. Simply guard with the element reference.

```svelte
<!-- src/lib/features/crypto-chart/components/PriceChart.svelte -->
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { date: Date; price: number }[];
  }

  let { data }: Props = $props();

  let svgEl: SVGSVGElement;

  $effect(() => {
    // Guard: Only need to check element existence
    // $effect does NOT run during SSR in Svelte 5
    if (!svgEl) return;

    // D3 visualization logic
    const svg = d3.select(svgEl);
    svg.selectAll('*').remove();
    // ... scales, axes, line path ...

    // Optional: cleanup function
    return () => {
      svg.selectAll('*').remove();
    };
  });
</script>

<svg bind:this={svgEl} class="w-full h-full" />
```

> **Note:** The `browser` check from `$app/environment` is only needed when you have code that runs *outside* of `$effect` (e.g., top-level module code). For `$derived` that references browser APIs like `window.innerWidth`, use a fallback pattern:
> ```typescript
> let width = $derived(typeof window !== 'undefined' ? window.innerWidth : 800);
> ```

## Loading States

Unlike React's Suspense, SvelteKit handles loading states through the routing system:

### Option 1: `+page.ts` with `await`

Data is loaded before the page renders. No loading spinner needed in the component.

### Option 2: Streaming with `+page.server.ts`

```typescript
// +page.server.ts
export const load = async () => {
  return {
    // Streamed promise - page renders immediately
    chartData: fetchCryptoHistory('BTC')
  };
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  let { data } = $props();
</script>

{#await data.chartData}
  <div class="skeleton h-[400px]" />
{:then chartData}
  <PriceChart data={chartData} />
{:catch error}
  <p class="text-red-500">Failed to load chart: {error.message}</p>
{/await}
```

## Dynamic Imports (Code Splitting)

If the D3 library is large or the chart is below the fold, lazy load the component using Svelte's `{#await}` pattern:

```svelte
<!-- Recommended: Clean and declarative -->
{#await import('./PriceChart.svelte')}
  <div class="skeleton h-[400px]" />
{:then { default: PriceChart }}
  <PriceChart {data} />
{:catch error}
  <p class="text-red-500">Failed to load chart</p>
{/await}
```

This pattern is simpler than the `onMount` + state variable approach and handles loading/error states declaratively.

## Key Differences from Next.js

| Aspect | Next.js | SvelteKit |
|--------|---------|-----------|
| Client Marking | `'use client'` directive | `$effect` with ref guard |
| Server Data | Server Component props | `+page.server.ts` → `data` prop |
| Suspense | `<Suspense fallback={...}>` | `{#await}` block |
| Dynamic Import | `next/dynamic` with `ssr: false` | Dynamic `import()` in `onMount` |
