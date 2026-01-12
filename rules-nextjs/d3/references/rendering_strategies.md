# Rendering Strategies (SVG vs Canvas)

## Rationale

Choose a rendering strategy based on the number of data points to balance performance and interactivity. SVG offers easier styling and event handling but degrades in performance with many elements. Canvas is pixel-based and optimized for large datasets.

## Decision Criteria

| Data Points | Strategy | Reason |
|----|-----|-----|
| **< 1,000** | **SVG Only** | No performance issues. Easy CSS styling and event handling. |
| **1,000 ~ 10,000** | **SVG+Canvas Hybrid** | SVG for axes/tooltips, Canvas for points. Good balance. |
| **> 10,000** | **Canvas Only (Hybrid)** | All elements via Canvas or Hybrid with heavy Canvas use. Interaction via Hit Detection. |

> **Note:** "Data points" refers to the number of elements simultaneously rendered. For time-series, calculate `Dates × Series`.

## Pattern A: SVG Only (Low Data Volume)

Best for simple charts or when CSS styling/animations are prioritized.

```typescript
'use client';

import { useRef, useLayoutEffect } from 'react';
import * as d3 from 'd3';

interface ChartProps {
  data: DataPoint[];
}

export function SimpleChart({ data }: ChartProps) {
  const svgRef = useRef<SVGSVGElement>(null);

  useLayoutEffect(() => {
    const svg = svgRef.current;
    if (!svg || data.length === 0) return;

    // 1. Reset
    svg.innerHTML = '';
    const width = 800;
    const height = 400;

    // 2. Scales
    const xScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([0, width]);

    const yScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([height, 0]);

    // 3. Axes
    d3.select(svg)
      .append('g')
      .attr('transform', `translate(0, ${height})`)
      .call(d3.axisBottom(xScale));

    d3.select(svg)
      .append('g')
      .call(d3.axisLeft(yScale));

    // 4. Data Points
    d3.select(svg)
      .selectAll('circle')
      .data(data)
      .enter()
      .append('circle')
      .attr('cx', d => xScale(d.x))
      .attr('cy', d => yScale(d.y))
      .attr('r', 4)
      .attr('fill', 'steelblue');
  }, [data]);

  return <svg ref={svgRef} width={800} height={400} />;
}
```

## Pattern B: SVG+Canvas Hybrid (High Data Volume)

The standard for performance-critical visualizations.
- **Canvas**: Renders 10k+ circles/lines effortlessly.
- **SVG**: Renders clear text, axes, and grids without pixelation.

```typescript
'use client';

import { useRef, useLayoutEffect } from 'react';
import * as d3 from 'd3';

export function HybridChart({ data }: { data: DataPoint[] }) {
  const svgRef = useRef<SVGSVGElement>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    const svg = svgRef.current;
    const canvas = canvasRef.current;
    const container = containerRef.current;
    
    if (!svg || !canvas || !container || data.length === 0) return;

    const width = container.clientWidth;
    const height = container.clientHeight;
    
    // 1. Setup Canvas
    canvas.width = width;
    canvas.height = height;
    
    // 2. Setup SVG
    svg.setAttribute('width', String(width));
    svg.setAttribute('height', String(height));
    svg.innerHTML = '';

    // 3. Scales
    const xScale = d3.scaleLinear().domain(...).range([0, width]);
    const yScale = d3.scaleLinear().domain(...).range([height, 0]);

    // 4. Render Axes (SVG)
    d3.select(svg)
      .append('g')
      .call(d3.axisBottom(xScale));

    // 5. Render Data Points (Canvas)
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    ctx.clearRect(0, 0, width, height);
    ctx.fillStyle = 'steelblue';

    data.forEach(d => {
      ctx.beginPath();
      // Draw point on canvas
      ctx.arc(xScale(d.x), yScale(d.y), 2, 0, 2 * Math.PI);
      ctx.fill();
    });
  }, [data]);

  return (
    <div ref={containerRef} className="relative w-full h-full">
      <canvas ref={canvasRef} className="absolute inset-0" />
      <svg ref={svgRef} className="absolute inset-0 pointer-events-none" />
    </div>
  );
}
```

## Anti-Patterns

- ❌ **SVG Overload**: Rendering > 3,000 DOM nodes with SVG. Causes layout thrashing and slow interactions.
- ❌ **Canvas for Everything**: Using Canvas for simple text/axes. Results in blurry text on high-DPI screens and hard accessibility implementation.
