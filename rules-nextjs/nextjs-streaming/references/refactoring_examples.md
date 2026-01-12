---
title: Refactoring Example
description: Before and After example of refactoring for streaming.
---

# Refactoring for Streaming

## Scenario: Momentum Analysis Dashboard

### ❌ Before: Monolithic & Blocking

```typescript
// Monolithic component fetches everything
export async function MomentumAnalysisContent({ searchParams }) {
  // Manual parsing
  const days = Number(searchParams.days || 20);
  
  // Sequential blocking fetch
  const [data, table, tickers] = await Promise.all([
    fetchChart(days),
    fetchTable(days),
    fetchTickers()
  ]);

  return (
    <>
      <Chart data={data} />
      <Table data={table} />
      <Tickers data={tickers} />
    </>
  );
}
```
*Result:* User waits for ALL data (chart + table + tickers) before seeing anything.

### ✅ After: Granular & Parallel

```typescript
// 1. Page Orchestration
export default async function Page({ searchParams }) {
  await cache.parse(searchParams); // nuqs cache
  
  return (
    <div className="space-y-6">
      <Suspense fallback={<ChartSkeleton />}>
        <ChartContainer />
      </Suspense>
      <Suspense fallback={<TableSkeleton />}>
        <TableContainer />
      </Suspense>
    </div>
  );
}

// 2. Granular Components
async function ChartContainer() {
  const { days } = cache.all(); // Read URL state
  const data = await fetchChart(days);
  const tickers = await fetchTickers(); // Parallel to Table fetch
  return <Chart data={data} tickers={tickers} />;
}

async function TableContainer() {
  const { days } = cache.all();
  const data = await fetchTable(days);
  return <Table data={data} />;
}
```
*Result:* Chart and Table stream independently. If Chart is fast, it shows up immediately while Table loads.
