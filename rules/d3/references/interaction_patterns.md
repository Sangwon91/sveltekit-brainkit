# Interaction Handling (Hit Detection vs Overlay)

## Overview

Handling interactions (hover, click, drag) with separate rendering layers requires a strategy to map mouse coordinates to data.

## Strategy A: Canvas Hit Detection

**Best for:** High performance, huge datasets (>10k points), simple hover/click.

**Mechanism:** 
1.  Listen to mouse events on the Canvas.
2.  Calculate mouse coordinates relative to the canvas.
3.  Inverse map coordinates to data or calculate Euclidean distance to nearest point.

**Svelte Implementation:**

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface DataPoint {
    x: number;
    y: number;
  }

  interface Props {
    data: DataPoint[];
  }

  let { data }: Props = $props();

  let canvasEl: HTMLCanvasElement;
  let hoveredPoint: DataPoint | null = $state(null);

  // Scales - will be set in $effect
  let xScale: d3.ScaleLinear<number, number>;
  let yScale: d3.ScaleLinear<number, number>;

  function findNearestPoint(mouseX: number, mouseY: number): DataPoint | null {
    if (!xScale || !yScale) return null;

    let minDistance = Infinity;
    let nearest: DataPoint | null = null;
    const threshold = 10; // pixels

    for (const d of data) {
      const px = xScale(d.x);
      const py = yScale(d.y);
      const dist = Math.sqrt((mouseX - px) ** 2 + (mouseY - py) ** 2);

      if (dist < threshold && dist < minDistance) {
        minDistance = dist;
        nearest = d;
      }
    }
    return nearest;
  }

  function handleMouseMove(event: MouseEvent) {
    const rect = canvasEl.getBoundingClientRect();
    const mouseX = event.clientX - rect.left;
    const mouseY = event.clientY - rect.top;
    hoveredPoint = findNearestPoint(mouseX, mouseY);
  }

  $effect(() => {
    // Setup scales and render...
    xScale = d3.scaleLinear().domain([0, 100]).range([0, 800]);
    yScale = d3.scaleLinear().domain([0, 100]).range([400, 0]);
    // ... rendering logic
  });
</script>

<canvas
  bind:this={canvasEl}
  onmousemove={handleMouseMove}
  onmouseleave={() => hoveredPoint = null}
/>

{#if hoveredPoint}
  <div class="tooltip">
    Point: ({hoveredPoint.x}, {hoveredPoint.y})
  </div>
{/if}
```

**Optimization:** Use a Quadtree (`d3-quadtree`) for O(log n) lookup if data is massive (>10k).

## Strategy B: SVG Invisible Overlay (Voronoi)

**Best for:** Complex interactions, best UX (fuzzy finding), moderate datasets (<2k points).

**Mechanism:**
1.  Create a transparent SVG layer on top (`z-index: highest`).
2.  Use `d3-delaunay` (Voronoi tessellation) to partition the space.
3.  Each data point gets a polygon region. Hovering anywhere in the region activates the point.

**Svelte Implementation:**

```svelte
<script lang="ts">
  import * as d3 from 'd3';
  import { Delaunay } from 'd3-delaunay';

  interface DataPoint {
    x: number;
    y: number;
  }

  interface Props {
    data: DataPoint[];
  }

  let { data }: Props = $props();

  let overlayEl: SVGSVGElement;
  let hoveredPoint: DataPoint | null = $state(null);

  $effect(() => {
    if (!overlayEl || data.length === 0) return;

    const width = 800;
    const height = 400;

    const xScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([0, width]);

    const yScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([height, 0]);

    const delaunay = Delaunay.from(
      data,
      d => xScale(d.x),
      d => yScale(d.y)
    );
    const voronoi = delaunay.voronoi([0, 0, width, height]);

    d3.select(overlayEl)
      .selectAll('path')
      .data(data)
      .join('path')
      .attr('d', (_, i) => voronoi.renderCell(i))
      .attr('fill', 'transparent')
      .attr('stroke', 'none') // Debug: 'red' to see regions
      .style('pointer-events', 'all')
      .on('mouseenter', (_, d) => { hoveredPoint = d; })
      .on('mouseleave', () => { hoveredPoint = null; });
  });
</script>

<svg bind:this={overlayEl} class="absolute inset-0" style="z-index: 100;" />
```

## Comparison

| Feature | Hit Detection (Canvas) | SVG Overlay (Voronoi) |
|---------|------------------------|-----------------------|
| **Perf** | Excellent | Good (< 2k nodes) |
| **UX** | Precise targeting required | Very forgiving (nearest neighbor) |
| **Memory** | Low | High (DOM nodes) |
| **Events** | `onmousemove` on canvas | Native DOM events on paths |

## Mixed Approach (Hidden Canvas / Color Picking)

For massive datasets where Voronoi is too slow but "color picking" hit detection is needed:
1.  Draw data to visible canvas.
2.  Draw data to an *off-screen* canvas using unique colors per data point (RGB values mapped to ID).
3.  On mousemove, read the pixel color at `(x, y)` from the off-screen canvas.
4.  Decode color back to Data ID.
5.  O(1) lookup.

This technique is especially powerful for complex shapes (polygons, paths) where Euclidean distance doesn't apply.
