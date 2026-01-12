# Rule 4: Test Depth - Solitary vs Sociable

### Two Approaches to Testing

Understanding when to use **Solitary** (isolated unit tests) vs **Sociable** (integration tests) is crucial for effective testing:

### Solitary (Isolated Unit Tests)

- **Scope:** Test a single class/function in isolation
- **Dependencies:** All dependencies are mocked/stubbed (but prefer real objects when possible - Chicago School)
- **Use Case:** Business logic, pure functions, algorithms
- **Speed:** Very fast
- **Example:** Testing a calculation function, validation logic, data transformation
- **Chicago School Connection:** Prefer state verification with real objects. Only mock when dependencies are slow/expensive.
- **London School Connection:** Can use behavior verification with mocks when interaction contracts are critical.

```python
# Python: Solitary test - Chicago School style (preferred)
def test_calculate_tax():
    # Pure function, no dependencies - use real implementation
    assert calculate_tax(100, rate=0.1) == 10
    assert calculate_tax(0, rate=0.1) == 0

# Python: Solitary test - London School style (when needed)
def test_process_order_with_mock(mocker):
    # Mock slow dependency, but use real domain objects when possible
    mock_payment = mocker.Mock()
    order = Order(items=[...])  # Real object
    result = process_order(order, payment_service=mock_payment)
    assert result.status == "processed"
    mock_payment.charge.assert_called_once()
```

```typescript
// Next.js: Solitary test - Chicago School style (preferred)
test('calculates tax', () => {
  // Pure function, no dependencies - use real implementation
  expect(calculateTax(100, 0.1)).toBe(10);
  expect(calculateTax(0, 0.1)).toBe(0);
});

// Next.js: Solitary test - London School style (when needed)
test('processes order with mock', () => {
  // Mock slow dependency, but use real domain objects when possible
  const mockPayment = { charge: vi.fn() };
  const order = new Order([...]); // Real object
  const result = processOrder(order, mockPayment);
  expect(result.status).toBe('processed');
  expect(mockPayment.charge).toHaveBeenCalledOnce();
});
```

### Sociable (Integration Tests)

- **Scope:** Test multiple components/modules working together
- **Dependencies:** Real dependencies (database, APIs) or test doubles that closely mimic real behavior
- **Use Case:** API endpoints, database operations, component interactions
- **Speed:** Slower but more realistic
- **Example:** Testing an API endpoint that queries a database, testing React component with real hooks
- **Chicago School Connection:** Prefer real objects and state verification. Use test doubles that closely mimic real behavior (in-memory DB, MSW).
- **London School Connection:** Can use mocks for external services, but prefer real test infrastructure when possible.

```python
# Python: Sociable test with real test database (Chicago School preferred)
def test_create_order_integration(db_session):
    # Uses real database (test database) - Chicago School approach
    # Real objects, real database, state verification
    order_data = {"user_id": 1, "items": [{"product_id": 1, "quantity": 2}]}
    order = create_order(order_data, db_session)
    
    # State verification - check final outcome
    assert order.id is not None
    assert order.status == "pending"
    
    # Verify in database - real integration
    saved_order = db_session.query(Order).filter_by(id=order.id).first()
    assert saved_order is not None
```

```typescript
// Next.js: Sociable test with MSW (Chicago School style)
// MSW mimics real API behavior, but we verify state, not protocol
import { render, screen } from '@testing-library/react';
import { server } from '@/test/mocks/server';
import { rest } from 'msw';

test('displays order list', async () => {
  // MSW provides realistic API behavior
  server.use(
    rest.get('/api/orders', (req, res, ctx) => {
      return res(ctx.json([{ id: 1, status: 'pending' }]));
    })
  );
  
  render(<OrderList />);
  
  // State verification - check what user sees
  expect(await screen.findByText('pending')).toBeInTheDocument();
});
```

### Best Practice: Hybrid Approach

- **Business Logic:** Use **Solitary** tests with **Chicago School** style (real objects, state verification). Only use mocks for slow/expensive dependencies.
- **API Endpoints:** Use **Sociable** tests that hit real test databases (**Chicago School** - real infrastructure, state verification)
- **UI Components:** Prefer **Sociable** tests that test user interactions (button clicks, form submissions) rather than isolated function calls (**Chicago School** - verify user-visible outcomes)
- **External Services:** Use **Chicago School** style (Mock as Stub, state verification) for external APIs, payment gateways, email services. Protocol compliance ensured by type/interface. Use **London School** (behavior verification) only when interaction verification is the test's primary concern.
