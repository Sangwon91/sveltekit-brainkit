# Interaction Handling (Hit Detection vs Overlay)

## Overview

Handling interactions (hover, click, drag) with separate rendering layers requires a strategy to map mouse coordinates to data.

## Strategy A: Canvas Hit Detection

**Best for:** High performance, huge datasets (>10k points), simple hover/click.

**Mechanism:** 
1.  Listen to mouse events on the Canvas.
2.  Calculate mouse coordinates relative to the canvas.
3.  Inverse map coordinates to data or calculate Euclidean distance to nearest point.

**Code Pattern:**

```typescript
const findNearestPoint = (mouseX, mouseY, data, xScale, yScale) => {
  // Simple Euclidean distance check
  let minDistance = Infinity;
  let nearest = null;
  const threshold = 10; // pixels

  for (const d of data) {
    const px = xScale(d.x);
    const py = yScale(d.y);
    const dist = Math.sqrt((mouseX - px)**2 + (mouseY - py)**2);
    
    if (dist < threshold && dist < minDistance) {
      minDistance = dist;
      nearest = d;
    }
  }
  return nearest;
};

// Optimization: use a Quadtree (d3-quadtree) for O(log n) lookup if data is massive (>10k)
```

## Strategy B: SVG Invisible Overlay (Voronoi)

**Best for:** Complex interactions, best UX (fuzzy finding), moderate datasets (<2k points).

**Mechanism:**
1.  Create a transparent SVG layer on top (`z-index: highest`).
2.  Use `d3-delaunay` (Voronoi tessellation) to partition the space.
3.  Each data point gets a polygon region. Hovering anywhere in the region activates the point.

**Code Pattern:**

```typescript
import { Delaunay } from 'd3-delaunay';

// Inside useLayoutEffect
const delaunay = Delaunay.from(data, d => xScale(d.x), d => yScale(d.y));
const voronoi = delaunay.voronoi([0, 0, width, height]);

d3.select(overlayRef.current)
  .selectAll('path')
  .data(data)
  .join('path')
  .attr('d', (d, i) => voronoi.renderCell(i))
  .attr('fill', 'transparent')
  .attr('stroke', 'none') // Debug: 'red' to see regions
  .style('pointer-events', 'all')
  .on('mousemove', (event, d) => setHovered(d));
```

## Comparison

| Feature | Hit Detection (Canvas) | SVG Overlay (Voronoi) |
|---------|------------------------|-----------------------|
| **Perf** | Excellent | Good (< 2k nodes) |
| **UX** | Precise targeting required | Very forgiving (nearest neighbor) |
| **Memory** | Low | High (DOM nodes) |
| **Events** | `mousemove` on canvas | Native DOM events |

## Mixed Approach (Hidden Canvas)

For massive datasets where Voronoi is too slow but "color picking" hit detection is needed:
1.  Draw data to visible canvas.
2.  Draw data to an *off-screen* canvas using unique colors per data point (e.g., RGB values mapped to ID).
3.  On mousemove, read the pixel color at `(x, y)` from the off-screen canvas.
4.  Decode color back to Data ID.
5.  O(1) lookup.
