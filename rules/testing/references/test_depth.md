# Rule 4: Test Depth - Solitary vs Sociable

## Two Approaches to Testing

Understanding when to use **Solitary** (isolated unit tests) vs **Sociable** (integration tests) is crucial for effective testing:

## Solitary (Isolated Unit Tests)

- **Scope:** Test a single class/function in isolation
- **Dependencies:** All dependencies are mocked/stubbed (but prefer real objects when possible - Chicago School)
- **Use Case:** Business logic, pure functions, algorithms
- **Speed:** Very fast
- **Example:** Testing a calculation function, validation logic, data transformation

### Python Example

```python
# Solitary test - Chicago School style (preferred)
def test_calculate_tax():
    # Pure function, no dependencies - use real implementation
    assert calculate_tax(100, rate=0.1) == 10
    assert calculate_tax(0, rate=0.1) == 0

# Solitary test - London School style (when needed)
def test_process_order_with_mock(mocker):
    # Mock slow dependency, but use real domain objects when possible
    mock_payment = mocker.Mock()
    order = Order(items=[...])  # Real object
    result = process_order(order, payment_service=mock_payment)
    assert result.status == "processed"
    mock_payment.charge.assert_called_once()
```

### SvelteKit Example

```typescript
// Solitary test - Chicago School style (preferred)
test('calculates tax', () => {
  // Pure function, no dependencies - use real implementation
  expect(calculateTax(100, 0.1)).toBe(10);
  expect(calculateTax(0, 0.1)).toBe(0);
});

// Solitary test - London School style (when needed)
test('processes order with mock', () => {
  // Mock slow dependency, but use real domain objects when possible
  const mockPayment = { charge: vi.fn() };
  const order = new Order([...]); // Real object
  const result = processOrder(order, mockPayment);
  expect(result.status).toBe('processed');
  expect(mockPayment.charge).toHaveBeenCalledOnce();
});
```

## Sociable (Integration Tests)

- **Scope:** Test multiple components/modules working together
- **Dependencies:** Real dependencies (database, APIs) or test doubles that closely mimic real behavior
- **Use Case:** API endpoints, database operations, component interactions
- **Speed:** Slower but more realistic
- **Example:** Testing an API endpoint that queries a database, testing Svelte component with real stores

### Python Example

```python
# Sociable test with real test database (Chicago School preferred)
def test_create_order_integration(db_session):
    # Uses real database (test database) - Chicago School approach
    order_data = {"user_id": 1, "items": [{"product_id": 1, "quantity": 2}]}
    order = create_order(order_data, db_session)
    
    # State verification - check final outcome
    assert order.id is not None
    assert order.status == "pending"
    
    # Verify in database - real integration
    saved_order = db_session.query(Order).filter_by(id=order.id).first()
    assert saved_order is not None
```

### SvelteKit Example

```typescript
// Sociable test with MSW (Chicago School style)
// MSW mimics real API behavior, but we verify state, not protocol
import { render, screen } from '@testing-library/svelte';
import { server } from '$lib/test/mocks/server';
import { http, HttpResponse } from 'msw';
import OrderList from './OrderList.svelte';

test('displays order list', async () => {
  // MSW provides realistic API behavior
  server.use(
    http.get('/api/orders', () => {
      return HttpResponse.json([{ id: 1, status: 'pending' }]);
    })
  );
  
  render(OrderList);
  
  // State verification - check what user sees
  expect(await screen.findByText('pending')).toBeInTheDocument();
});
```

### SvelteKit Component Test with Svelte Testing Library

```typescript
// Testing Svelte component with user interactions
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import Counter from './Counter.svelte';

test('increments count on button click', async () => {
  const user = userEvent.setup();
  render(Counter, { props: { initialCount: 0 } });
  
  const button = screen.getByRole('button', { name: /increment/i });
  await user.click(button);
  
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

## Best Practice: Hybrid Approach

| Test Type | Style | When to Use |
|-----------|-------|-------------|
| **Business Logic** | Solitary + Chicago | Real objects, state verification. Only mock slow/expensive dependencies. |
| **API Endpoints** | Sociable + Chicago | Real test databases, state verification |
| **UI Components** | Sociable + Chicago | Test user interactions (clicks, inputs), verify user-visible outcomes |
| **External Services** | Solitary + Chicago | Use Fakes for state verification. Use London style only when interaction is the primary concern. |

## SvelteKit Testing Setup

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte({ hot: false })],
  test: {
    include: ['src/**/*.{test,spec}.{js,ts}'],
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/lib/test/setup.ts'],
  },
});
```

```typescript
// src/lib/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```
