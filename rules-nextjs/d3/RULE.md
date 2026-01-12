---
alwaysApply: false
---

# D3.js Visualization Rules

## Core Philosophy

-   **Pure D3.js Only:** Use D3.js directly without wrapper libraries (visx, react-d3). Embrace the imperative API for maximum control.
-   **Performance-First:** Explicitly choose between SVG and Canvas based on data volume.
-   **Layer Separation:** Use the "Block Box" pattern with separate refs for SVG (axes) and Canvas (data) to isolate responsibilities.
-   **Lifecycle-Aware:** Reconcile React's declarative nature with D3's imperative updates using `useLayoutEffect`.

## 1. Rendering Strategy

Choose the rendering backend based on the number of simulatenous elements.

| Data Points | Strategy | Note |
|---|---|---|
| **< 1,000** | **SVG Only** | Simplest. CSS styling and native events work out of the box. |
| **1k - 10k** | **Hybrid** | SVG for axes/text, Canvas for data points. |
| **> 10,000** | **Canvas** | Full Canvas or Hybrid. Interaction requires hit detection. |

-   See [Rendering Strategies Details](references/rendering_strategies.md) for implementations.

## 2. Layer Architecture (Block Box)

Prevent DOM conflicts by giving D3 its own sandbox via refs.

-   **Structure**: Container `div` (relative) -> `canvas` (absolute) -> `svg` (absolute, pointer-events-none).
-   **Constraint**: React manages the container and refs. D3 manages everything *inside* the refs.
-   **Cleanup**: Always clear the specific layer (`innerHTML = ''` or `clearRect`) before re-drawing in `useLayoutEffect`.

-   See [Layer Architecture](references/layer_architecture.md) for the boilerplate.

## 3. Interaction Strategy

-   **Small Data (< 2k)**: Use **SVG Overlay** (Voronoi or invisible circles) for fuzzy finding and best UX.
-   **Large Data (> 2k)**: Use **Canvas Hit Detection** (Euclidean distance or Color-picking) for performance.

-   See [Interaction Patterns](references/interaction_patterns.md).

## 4. React Integration Workflow

1.  **Mark as Client Component**: Add `'use client'` to all chart files.
2.  **Use `useLayoutEffect`**: Prevents "flash of unstyled chart" by rendering synchronously before paint.
3.  **Responsive Container**: Use a `ResizeObserver` pattern on the parent `div` to drive D3 dimensions.
4.  **Dependencies**: Include all data and config arrays in the effect dependency list. Memoize expensive data transforms outside the effect.
5.  **Server Boundary**: Fetch data in Server Components and pass down as props. Wrap in `<Suspense>`.

-   See [Next.js Integration](references/nextjs_integration.md) and [Optimization](references/optimization.md).

## Example: Hybrid Chart Template

```typescript
'use client';
import { useRef, useLayoutEffect } from 'react';
import * as d3 from 'd3';

export function HybridChart({ data }) {
  const containerRef = useRef(null);
  const svgRef = useRef(null);      // Axis Layer
  const canvasRef = useRef(null);   // Data Layer

  useLayoutEffect(() => {
    if (!containerRef.current) return;
    const { width, height } = containerRef.current.getBoundingClientRect();

    // 1. Setup Canvas
    const ctx = canvasRef.current.getContext('2d');
    canvasRef.current.width = width;
    canvasRef.current.height = height;
    ctx.clearRect(0, 0, width, height);

    // 2. Setup SVG
    const svg = d3.select(svgRef.current);
    svg.attr('width', width).attr('height', height);
    svg.selectAll('*').remove();

    // 3. Draw (D3 Logic)
    // ... scales, axes, points ...
    
  }, [data]); // Add config deps here

  return (
    <div ref={containerRef} className="relative w-full h-[400px]">
      <canvas ref={canvasRef} className="absolute inset-0" />
      <svg ref={svgRef} className="absolute inset-0 pointer-events-none" />
    </div>
  );
}
```
