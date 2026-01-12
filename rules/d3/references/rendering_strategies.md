# Rendering Strategies (SVG vs Canvas)

## Rationale

Choose a rendering strategy based on the number of data points to balance performance and interactivity. SVG offers easier styling and event handling but degrades in performance with many elements. Canvas is pixel-based and optimized for large datasets.

## Decision Criteria

| Data Points | Strategy | Reason |
|----|-----|-----|
| **< 1,000** | **SVG Only** | No performance issues. Easy CSS styling and event handling. |
| **1,000 ~ 10,000** | **SVG+Canvas Hybrid** | SVG for axes/tooltips, Canvas for points. Good balance. |
| **> 10,000** | **Canvas Only (Hybrid)** | All elements via Canvas or Hybrid with heavy Canvas use. |

> **Note:** "Data points" refers to the number of elements simultaneously rendered. For time-series, calculate `Dates × Series`.

## Pattern A: SVG Only (Low Data Volume)

Best for simple charts or when CSS styling/animations are prioritized.

### Option 1: D3 Imperative (Complex charts, animations, axes)

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  let svgEl: SVGSVGElement;

  const width = 800;
  const height = 400;
  const margin = { top: 20, right: 20, bottom: 30, left: 40 };

  $effect(() => {
    if (!svgEl || data.length === 0) return;

    const svg = d3.select(svgEl);
    svg.selectAll('*').remove();

    const innerWidth = width - margin.left - margin.right;
    const innerHeight = height - margin.top - margin.bottom;

    // Scales
    const xScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([0, innerWidth]);

    const yScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([innerHeight, 0]);

    // Group for margins
    const g = svg.append('g')
      .attr('transform', `translate(${margin.left}, ${margin.top})`);

    // Axes
    g.append('g')
      .attr('transform', `translate(0, ${innerHeight})`)
      .call(d3.axisBottom(xScale));

    g.append('g')
      .call(d3.axisLeft(yScale));

    // Data Points
    g.selectAll('circle')
      .data(data)
      .join('circle')
      .attr('cx', d => xScale(d.x))
      .attr('cy', d => yScale(d.y))
      .attr('r', 4)
      .attr('fill', 'steelblue');

    // Cleanup on unmount or re-render
    return () => svg.selectAll('*').remove();
  });
</script>

<svg bind:this={svgEl} {width} {height} />
```

### Option 2: Svelte Declarative (Simple charts, no axes)

For very simple visualizations without D3 axes, consider letting Svelte manage the SVG elements:

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  const width = 800;
  const height = 400;

  // D3 for calculations only
  let xScale = $derived(
    d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([40, width - 20])
  );

  let yScale = $derived(
    d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([height - 30, 20])
  );
</script>

<!-- Svelte manages DOM, D3 provides scales -->
<svg {width} {height}>
  {#each data as d}
    <circle
      cx={xScale(d.x)}
      cy={yScale(d.y)}
      r={4}
      fill="steelblue"
      class="transition-all hover:fill-orange-500"
    />
  {/each}
</svg>
```

> **When to use which?**
> - **D3 Imperative (Recommended)**: Axes, transitions, complex interactions, brushing, zooming. **Use this by default for consistency.**
> - **Svelte Declarative (Exception)**: Very simple shapes without axes, when CSS hover effects are the only interaction needed.

## Pattern B: SVG+Canvas Hybrid (High Data Volume)

The standard for performance-critical visualizations.
- **Canvas**: Renders 10k+ circles/lines effortlessly.
- **SVG**: Renders clear text, axes, and grids without pixelation.

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  let containerEl: HTMLDivElement;
  let svgEl: SVGSVGElement;
  let canvasEl: HTMLCanvasElement;

  $effect(() => {
    if (!containerEl || !svgEl || !canvasEl || data.length === 0) return;

    const { width, height } = containerEl.getBoundingClientRect();
    const margin = { top: 20, right: 20, bottom: 30, left: 40 };

    // 1. Setup Canvas
    canvasEl.width = width;
    canvasEl.height = height;

    // 2. Setup SVG
    svgEl.setAttribute('width', String(width));
    svgEl.setAttribute('height', String(height));
    svgEl.innerHTML = '';

    // 3. Scales
    const xScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.x) as [number, number])
      .range([margin.left, width - margin.right]);

    const yScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.y) as [number, number])
      .range([height - margin.bottom, margin.top]);

    // 4. Render Axes (SVG)
    const svg = d3.select(svgEl);
    
    svg.append('g')
      .attr('transform', `translate(0, ${height - margin.bottom})`)
      .call(d3.axisBottom(xScale));

    svg.append('g')
      .attr('transform', `translate(${margin.left}, 0)`)
      .call(d3.axisLeft(yScale));

    // 5. Render Data Points (Canvas)
    const ctx = canvasEl.getContext('2d');
    if (!ctx) return;

    ctx.clearRect(0, 0, width, height);
    ctx.fillStyle = 'steelblue';

    for (const d of data) {
      ctx.beginPath();
      ctx.arc(xScale(d.x), yScale(d.y), 2, 0, 2 * Math.PI);
      ctx.fill();
    }
  });
</script>

<div bind:this={containerEl} class="relative w-full h-full min-h-[400px]">
  <canvas bind:this={canvasEl} class="absolute inset-0" />
  <svg bind:this={svgEl} class="absolute inset-0 pointer-events-none" />
</div>
```

## Svelte Advantages

Unlike React, Svelte's `$effect` runs synchronously after DOM updates, eliminating the "flash of empty content" issue that requires `useLayoutEffect` in React. D3's direct DOM manipulation integrates naturally without VDOM conflicts.

## Anti-Patterns

- ❌ **SVG Overload**: Rendering > 3,000 DOM nodes with SVG. Causes layout thrashing and slow interactions.
- ❌ **Canvas for Everything**: Using Canvas for text/axes. Results in blurry text on high-DPI screens.
- ❌ **Missing Cleanup**: Always clear previous content with `innerHTML = ''` or `ctx.clearRect()` before re-drawing.
