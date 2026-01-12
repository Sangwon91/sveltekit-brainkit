# Re-rendering Strategy & Optimization

## Rationale

Unlike React, Svelte doesn't have a Virtual DOM that conflicts with D3's imperative approach. However, we still need to define exactly *when* the chart runs its update logic to avoid unnecessary re-renders.

## Re-render Triggers

D3 logic inside `$effect` should run when:
1.  **Data Changes** (`data` prop).
2.  **Dimensions Change** (Container resize).
3.  **Config Changes** (e.g., toggling Log/Linear scale).

It should **NOT** run when:
1.  Tooltip state changes (handled via separate Svelte state).
2.  Unrelated parent component updates (Svelte handles this automatically).

## Svelte 5 Runes Advantages

Svelte's fine-grained reactivity means `$effect` only runs when its tracked dependencies change. No manual dependency arrays needed.

```svelte
<script lang="ts">
  let { data, config } = $props();
  let svgEl: SVGSVGElement;

  // $effect automatically tracks `data` and `config`
  $effect(() => {
    if (!svgEl) return;
    // Only re-runs when data or config actually changes
    renderChart(svgEl, data, config);
  });
</script>
```

## Responsive Containers

Charts should almost always fill their parent container.

### Pattern: ResizeObserver with `$effect`

This pattern makes the chart responsive to any layout change (sidebar toggle, window resize).

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  let containerEl: HTMLDivElement;
  let width = $state(0);
  let height = $state(0);

  // Resize observer effect
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

  // Render effect - reacts to data, width, height changes
  $effect(() => {
    if (width === 0 || height === 0) return;

    // D3 Logic using reactive `width` and `height`
    console.log(`Rendering at ${width}x${height} with ${data.length} points`);
  });
</script>

<div bind:this={containerEl} class="w-full h-full min-h-[400px]">
  <!-- Chart elements -->
</div>
```

## Derived Values for Expensive Calculations

Use `$derived` for expensive D3 calculations (layouts, hierarchies, path generations) to avoid recalculating on every render.

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: TreeNode;
    width: number;
    height: number;
  }

  let { data, width, height }: Props = $props();

  // ✅ Good: Memoize expensive hierarchy calculation
  let rootHierarchy = $derived(
    d3.hierarchy(data).sum(d => d.value)
  );

  // ✅ Good: Memoize scales
  let xScale = $derived(
    d3.scaleLinear()
      .domain([0, d3.max(data.children, d => d.x) ?? 100])
      .range([0, width])
  );

  let yScale = $derived(
    d3.scaleLinear()
      .domain([0, d3.max(data.children, d => d.y) ?? 100])
      .range([height, 0])
  );

  // Effect only runs when derived values or other deps change
  $effect(() => {
    // Use rootHierarchy, xScale, yScale for rendering
  });
</script>
```

## Separating Concerns: Data vs Interaction

Keep tooltip/hover state separate from the main render effect to avoid unnecessary full redraws:

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: DataPoint[];
  }

  let { data }: Props = $props();

  let svgEl: SVGSVGElement;
  let hoveredPoint: DataPoint | null = $state(null);

  // Main render - only runs when data changes
  $effect(() => {
    if (!svgEl) return;
    
    const svg = d3.select(svgEl);
    svg.selectAll('*').remove();
    renderChart(svg, data);

    // Cleanup: runs before next effect or on unmount
    return () => {
      svg.selectAll('*').remove();
    };
  });

  // Tooltip is handled separately via Svelte reactivity
  // Changing hoveredPoint does NOT trigger chart re-render
</script>

<svg bind:this={svgEl} />

{#if hoveredPoint}
  <div class="tooltip">
    {hoveredPoint.label}: {hoveredPoint.value}
  </div>
{/if}
```

## Anti-Patterns

```svelte
<!-- ❌ BAD: Heavy computation inside $effect -->
<script>
  $effect(() => {
    const hugeData = processMillionRows(data); // Blocks UI thread!
    renderChart(hugeData);
  });
</script>

<!-- ✅ GOOD: Extract to $derived -->
<script>
  let processedData = $derived(processMillionRows(data));

  $effect(() => {
    renderChart(processedData);
  });
</script>
```

```svelte
<!-- ❌ BAD: Missing cleanup -->
<script>
  $effect(() => {
    d3.select(svgEl).append('g'); // Appends NEW g every render!
  });
</script>

<!-- ✅ GOOD: Clear before render -->
<script>
  $effect(() => {
    d3.select(svgEl).selectAll('*').remove();
    d3.select(svgEl).append('g');
  });
</script>
```

## Performance Checklist

- [ ] Are expensive calculations in `$derived` instead of `$effect`?
- [ ] Is the chart using `ResizeObserver` for responsive sizing?
- [ ] Is cleanup happening in the `$effect` return function?
- [ ] Is hover/tooltip state separate from the main data render?
- [ ] Are you using Canvas for > 1000 data points?
- [ ] Is `$effect` guarding with just element refs?

## See Also

- [Canvas Patterns](canvas_patterns.md) for detailed Canvas rendering and performance tips
