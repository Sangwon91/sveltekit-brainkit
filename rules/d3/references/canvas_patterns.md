# Canvas Rendering Patterns

## Overview

Canvas is essential for rendering large datasets (1k+ points) without DOM performance degradation. This document covers patterns for pure Canvas and hybrid Canvas+SVG rendering.

## When to Use Canvas

| Scenario | Recommendation |
|----------|----------------|
| < 1,000 points | SVG is simpler |
| 1,000 - 10,000 points | Hybrid (Canvas data + SVG axes) |
| > 10,000 points | Pure Canvas or Hybrid |
| Animation (many frames) | Canvas preferred |
| High-DPI text/axes | SVG preferred (Canvas text is blurry) |

## Pattern A: Pure Canvas (10k+ Points)

Best for maximum performance with large datasets.

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface DataPoint {
    x: number;
    y: number;
    category: string;
  }

  interface Props {
    data: DataPoint[];
  }

  let { data }: Props = $props();

  let containerEl: HTMLDivElement;
  let canvasEl: HTMLCanvasElement;
  let width = $state(0);
  let height = $state(0);

  const margin = { top: 20, right: 20, bottom: 30, left: 40 };

  // Color mapping
  const colorMap: Record<string, string> = {
    A: '#3b82f6',
    B: '#ef4444',
    C: '#22c55e'
  };

  // Memoized scales
  let xScale = $derived(
    d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([margin.left, width - margin.right])
  );

  let yScale = $derived(
    d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([height - margin.bottom, margin.top])
  );

  // ResizeObserver effect
  $effect(() => {
    if (!containerEl) return;

    const observer = new ResizeObserver(entries => {
      const entry = entries[0];
      if (entry) {
        width = entry.contentRect.width;
        height = entry.contentRect.height;
      }
    });

    observer.observe(containerEl);
    return () => observer.disconnect();
  });

  // Render effect
  $effect(() => {
    if (!canvasEl || width === 0 || height === 0) return;

    const ctx = canvasEl.getContext('2d');
    if (!ctx) return;

    // HiDPI support
    const dpr = window.devicePixelRatio || 1;
    canvasEl.width = width * dpr;
    canvasEl.height = height * dpr;
    canvasEl.style.width = `${width}px`;
    canvasEl.style.height = `${height}px`;
    ctx.scale(dpr, dpr);

    // Clear
    ctx.clearRect(0, 0, width, height);

    // Background
    ctx.fillStyle = '#f8fafc';
    ctx.fillRect(0, 0, width, height);

    // Grid lines (simple Canvas grid)
    ctx.strokeStyle = '#e2e8f0';
    ctx.lineWidth = 1;
    for (let i = 0; i <= 10; i++) {
      const x = margin.left + (width - margin.left - margin.right) * i / 10;
      const y = margin.top + (height - margin.top - margin.bottom) * i / 10;
      
      // Vertical
      ctx.beginPath();
      ctx.moveTo(x, margin.top);
      ctx.lineTo(x, height - margin.bottom);
      ctx.stroke();
      
      // Horizontal
      ctx.beginPath();
      ctx.moveTo(margin.left, y);
      ctx.lineTo(width - margin.right, y);
      ctx.stroke();
    }

    // Data points (fast loop)
    for (const d of data) {
      ctx.beginPath();
      ctx.arc(xScale(d.x), yScale(d.y), 3, 0, Math.PI * 2);
      ctx.fillStyle = colorMap[d.category] ?? '#888';
      ctx.fill();
    }

    // Axes labels (Canvas text - acceptable for simple labels)
    ctx.fillStyle = '#374151';
    ctx.font = '12px system-ui, sans-serif';
    ctx.textAlign = 'center';
    
    // X-axis ticks
    const xTicks = xScale.ticks(5);
    for (const tick of xTicks) {
      ctx.fillText(String(tick), xScale(tick), height - 10);
    }
    
    // Y-axis ticks
    ctx.textAlign = 'right';
    const yTicks = yScale.ticks(5);
    for (const tick of yTicks) {
      ctx.fillText(String(tick), margin.left - 5, yScale(tick) + 4);
    }
  });
</script>

<div bind:this={containerEl} class="relative w-full h-[400px]">
  <canvas bind:this={canvasEl} class="absolute inset-0" />
</div>
```

## Pattern B: Hybrid Canvas+SVG

Best balance of performance and visual quality. Canvas for data, SVG for crisp text/axes.

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  let containerEl: HTMLDivElement;
  let canvasEl: HTMLCanvasElement;
  let svgEl: SVGSVGElement;

  const margin = { top: 20, right: 20, bottom: 30, left: 40 };

  $effect(() => {
    if (!containerEl || !canvasEl || !svgEl) return;

    const { width, height } = containerEl.getBoundingClientRect();
    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    // Scales
    const xScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([innerHeight, 0]);

    // ━━━━━━━━━━━━━━━━━━━━
    // 1. CANVAS: Data Points
    // ━━━━━━━━━━━━━━━━━━━━
    const dpr = window.devicePixelRatio || 1;
    canvasEl.width = width * dpr;
    canvasEl.height = height * dpr;
    canvasEl.style.width = `${width}px`;
    canvasEl.style.height = `${height}px`;

    const ctx = canvasEl.getContext('2d');
    if (!ctx) return;
    ctx.scale(dpr, dpr);
    ctx.clearRect(0, 0, width, height);

    // Translate for margins
    ctx.save();
    ctx.translate(margin.left, margin.top);

    ctx.fillStyle = 'steelblue';
    for (const d of data) {
      ctx.beginPath();
      ctx.arc(xScale(d.x), yScale(d.y), 3, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.restore();

    // ━━━━━━━━━━━━━━━━━━━━
    // 2. SVG: Axes (crisp text)
    // ━━━━━━━━━━━━━━━━━━━━
    const svg = d3.select(svgEl);
    svg.attr('width', width).attr('height', height);
    svg.selectAll('*').remove();

    const g = svg.append('g')
      .attr('transform', `translate(${margin.left}, ${margin.top})`);

    // X Axis
    g.append('g')
      .attr('transform', `translate(0, ${innerHeight})`)
      .call(d3.axisBottom(xScale));

    // Y Axis
    g.append('g')
      .call(d3.axisLeft(yScale));

    // Cleanup
    return () => {
      svg.selectAll('*').remove();
      ctx.clearRect(0, 0, width, height);
    };
  });
</script>

<div bind:this={containerEl} class="relative w-full h-[400px]">
  <!-- Canvas: lower z-index (back) -->
  <canvas bind:this={canvasEl} class="absolute inset-0" style="z-index: 10;" />
  <!-- SVG: higher z-index (front), pointer-events-none for click-through -->
  <svg bind:this={svgEl} class="absolute inset-0 pointer-events-none" style="z-index: 20;" />
</div>
```

## HiDPI Support

Always handle high-DPI displays for sharp rendering:

```typescript
// Setup HiDPI canvas
const dpr = window.devicePixelRatio || 1;
canvas.width = width * dpr;
canvas.height = height * dpr;
canvas.style.width = `${width}px`;
canvas.style.height = `${height}px`;

const ctx = canvas.getContext('2d');
ctx.scale(dpr, dpr);

// Now draw at logical pixels - Canvas handles the scaling
ctx.arc(100, 100, 5, 0, Math.PI * 2); // Draws at 100,100 logical pixels
```

## Performance Tips

### 1. Batch Similar Operations

```typescript
// ❌ Slow: Switching fill color per point
for (const d of data) {
  ctx.fillStyle = colorMap[d.category];
  ctx.fillRect(xScale(d.x), yScale(d.y), 4, 4);
}

// ✅ Fast: Group by color, batch draws
const grouped = d3.group(data, d => d.category);
for (const [category, points] of grouped) {
  ctx.fillStyle = colorMap[category];
  for (const d of points) {
    ctx.fillRect(xScale(d.x), yScale(d.y), 4, 4);
  }
}
```

### 2. Use requestAnimationFrame for Animation

```typescript
let animationId: number;

$effect(() => {
  if (!canvasEl) return;
  
  function animate() {
    renderFrame();
    animationId = requestAnimationFrame(animate);
  }
  
  animate();
  
  return () => cancelAnimationFrame(animationId);
});
```

### 3. Offscreen Canvas for Complex Rendering

```typescript
// Pre-render complex shapes to offscreen canvas
const offscreen = new OffscreenCanvas(32, 32);
const offCtx = offscreen.getContext('2d');
offCtx.arc(16, 16, 14, 0, Math.PI * 2);
offCtx.fill();

// Then stamp it many times
for (const d of data) {
  ctx.drawImage(offscreen, xScale(d.x) - 16, yScale(d.y) - 16);
}
```

## Anti-Patterns

| ❌ Anti-Pattern | ✅ Correct Approach |
|-----------------|---------------------|
| Canvas for text/axes | Use SVG for crisp text |
| Missing HiDPI handling | Always use `devicePixelRatio` |
| Recreating canvas context each frame | Cache context reference |
| Drawing outside visible area | Clip to viewport |
| Not clearing before redraw | Always `clearRect` first |
