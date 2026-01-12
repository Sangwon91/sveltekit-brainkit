# Layer Separation Architecture (Block Box Pattern)

## Rationale

When mixing SVG and Canvas, or when managing complex chart layers, preventing DOM conflicts and keeping responsibilities clear is critical. We use the **Block Box Approach** where `bind:this` elements act as boundaries for D3 to take full control.

## Core Principles

1.  **Independent Bindings**: Each layer (SVG, Canvas) has its own `bind:this` variable. Svelte never touches the *contents*, only the *container*.
2.  **Delegation to D3**: Inside `$effect`, D3 takes full ownership of the bound element's content.
3.  **Visual Stacking**: Use CSS `position: absolute` and `z-index` to stack layers.

## Layer Responsibilities

| Layer | Technology | Z-Index | Responsibility |
|-------|------------|---------|----------------|
| **Interaction** | SVG or HTML (Div) | 100 | Captures mouse events, displays tooltips |
| **Overlay** | SVG | 50 | Crosshairs, selection box, annotations |
| **Grid/Axes** | SVG | 20 | Axes, grid lines, labels (crisp text) |
| **Data** | Canvas | 10 | High-volume data points (heatmap, scatter) |
| **Background** | SVG/Canvas | 0 | Chart background color/pattern |

## Implementation Pattern

```svelte
<script lang="ts">
  import * as d3 from 'd3';

  interface Props {
    data: { x: number; y: number }[];
  }

  let { data }: Props = $props();

  // 1. Separate Bindings
  let containerEl: HTMLDivElement;
  let svgEl: SVGSVGElement;      // Axes/Static
  let canvasEl: HTMLCanvasElement; // Data

  $effect(() => {
    if (!containerEl || !svgEl || !canvasEl) return;

    const { width, height } = containerEl.getBoundingClientRect();

    // Layer 1: SVG Control
    const svg = d3.select(svgEl);
    svg.attr('width', width).attr('height', height);
    svg.selectAll('*').remove(); // Clear previous
    // Draw axes...

    // Layer 2: Canvas Control
    canvasEl.width = width;
    canvasEl.height = height;
    const ctx = canvasEl.getContext('2d');
    if (!ctx) return;
    ctx.clearRect(0, 0, width, height); // Clear previous
    // Draw data...
  });
</script>

<div bind:this={containerEl} class="relative w-full h-full">
  <!-- Visual Stacking Order -->
  <canvas
    bind:this={canvasEl}
    class="absolute inset-0"
    style="z-index: 10;"
  />
  <svg
    bind:this={svgEl}
    class="absolute inset-0 pointer-events-none"
    style="z-index: 20;"
  />
</div>
```

## Svelte-Specific Benefits

1. **No Ref Boilerplate**: `bind:this` is simpler than React's `useRef` pattern.
2. **Reactive Cleanup**: `$effect` automatically tracks dependencies; manual dependency arrays aren't needed.
3. **Clean Transitions**: Svelte's built-in transitions can complement D3 animations.

## Checklist

- [ ] Does every layer have its own `bind:this` variable?
- [ ] Is CSS used to stack them (`absolute`, `inset-0`)?
- [ ] Is `pointer-events: none` applied to interaction-blocking overlay layers?
- [ ] Is cleanup handled via `$effect` return function?
- [ ] Are expensive calculations extracted to `$derived` outside `$effect`?
- [ ] Is `$effect` guarding with just element refs?

## See Also

- [Canvas Patterns](canvas_patterns.md) for detailed Canvas rendering examples
