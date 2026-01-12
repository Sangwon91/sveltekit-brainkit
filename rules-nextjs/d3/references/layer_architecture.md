# Layer Separation Architecture (Block Box Pattern)

## Rationale

When mixing SVG and Canvas, or when managing complex chart layers, prevent DOM conflicts and keeping responsibilities clear is critical. We use the **Block Box Approach** where `refs` act as boundaries for D3 to take full control.

## Core Principles

1.  **Independent Refs**: Each layer (SVG, Canvas) has its own `useRef`. React never touches the *contents*, only the *container*.
2.  **Delegation to D3**: Inside `useLayoutEffect`, D3 takes full ownership of the ref's content.
3.  **Visual Stacking**: Use CSS `position: absolute` and `z-index` to stack layers.

## Layer Responsibilities

| Layer | Technology | Z-Index | Responsibility |
|-------|------------|---------|----------------|
| **Interaction** | SVG or HTML (Div) | 100 | Captures mouse events, displays tooltips (optional) |
| **Overlay** | SVG | 50 | Crosshairs, selection box, annotations |
| **Grid/Axes** | SVG | 20 | Axes, grid lines, labels (crisp text) |
| **Data** | Canvas | 10 | High-volume data points (heatmap, scatter) |
| **Background** | SVG/Canvas | 0 | Chart background color/pattern |

## Implementation Pattern

```typescript
'use client';

import { useRef, useLayoutEffect } from 'react';
import * as d3 from 'd3';

export function LayeredChart({ data }) {
  // 1. Separate Refs
  const svgRef = useRef<SVGSVGElement>(null);      // Axes/Startic
  const canvasRef = useRef<HTMLCanvasElement>(null); // Data
  const containerRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    // ... setup dimensions ...

    // Layer 1: SVG Control
    const svg = d3.select(svgRef.current);
    svg.selectAll('*').remove(); // Clear previous
    // Draw axes...

    // Layer 2: Canvas Control
    const ctx = canvasRef.current.getContext('2d');
    ctx.clearRect(0, 0, width, height); // Clear previous
    // Draw data...

  }, [data]);

  return (
    <div ref={containerRef} className="relative w-full h-full">
      {/* Visual Stacking Order */}
      <canvas 
        ref={canvasRef} 
        className="absolute inset-0" 
        style={{ zIndex: 10 }} // Bottom
      />
      <svg 
        ref={svgRef} 
        className="absolute inset-0 pointer-events-none" 
        style={{ zIndex: 20 }} // Top
      />
    </div>
  );
}
```

## Checklist

- [ ] Does every layer have its own `ref`?
- [ ] Is CSS used to stack them (`absolute`, `top-0`, `left-0`)?
- [ ] Is `pointer-events: none` applied to interaction-blocking overlay layers (like the SVG axis layer) if they sit on top of interactive elements?
- [ ] Is cleanup handled for each layer explicitly?
