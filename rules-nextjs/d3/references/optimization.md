# Re-rendering Strategy & Optimization

## Rational

Reconciling React's declarative nature (VDOM) with D3's imperative nature (Direct DOM) is the source of most bugs. We must define exactly *when* the chart runs its update logic.

## Re-render Triggers

D3 logic inside `useLayoutEffect` should run when:
1.  **Data Changes** (`data` prop).
2.  **Dimensions Change** (Container resize).
3.  **Config Changes** (e.g., toggling Log/Linear scale).

It should **NOT** run when:
1.  Tooltip state changes (handled via separate React state/layer).
2.  Unrelated parent updates.

## Hooks: `useLayoutEffect` vs `useEffect`

Always use `useLayoutEffect` for D3 rendering.

-   **`useLayoutEffect`**: Runs synchronously after DOM mutations but *before* paint. Prevents visual flickering where the user sees the empty SVG before the chart appears.
-   **`useEffect`**: Runs after paint. Can cause specific "flash of empty content".

## Responsive Containers

Charts should almost always fill their parent container.

### Pattern 1: ResizeObserver (Simple & Robust)

This pattern makes the chart responsive to any layout change (sidebar toggle, window resize).

```typescript
export function ResponsiveChart({ data }) {
  const containerRef = useRef(null);
  const [size, setSize] = useState({ width: 0, height: 0 });

  useLayoutEffect(() => {
    const obs = new ResizeObserver(entries => {
      const entry = entries[0];
      if (entry) {
        const { width, height } = entry.contentRect;
        setSize({ width, height });
      }
    });
    obs.observe(containerRef.current);
    return () => obs.disconnect();
  }, []);

  useLayoutEffect(() => {
    if (size.width === 0) return;
    // D3 Logic here using `size.width` / `size.height`
  }, [data, size]); // Run when size changes

  return <div ref={containerRef} className="w-full h-full" />;
}
```

## Memoization

Expensive d3 calculations (like layouts, hierarchies, or path generations) should be memoized outside the effect.

```typescript
// ✅ Good: Memoize expensive data transform
const rootHierarchy = useMemo(() => {
  return d3.hierarchy(data).sum(d => d.value);
}, [data]);

// ✅ Good: Memoize scales if they are dependencies for other hooks
const xScale = useMemo(() => {
   return d3.scaleLinear().domain(...).range(...);
}, [data, width]);
```

## Anti-Patterns

```typescript
// ❌ BAD: Re-creating heavy objects inside the effect
useLayoutEffect(() => {
  const hugeData = processMillionRows(data); // Blocks UI thread on every render!
  // ...
}, [data]);

// ❌ BAD: Missing cleanup
useLayoutEffect(() => {
  d3.select(ref).append('g'); // Appends NEW g every render!
  // Fix: d3.select(ref).selectAll('*').remove();
}, []);
```
