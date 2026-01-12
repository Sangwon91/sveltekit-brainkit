# Rule 2: Testability Through Dependency Injection

## The Core Principle

**Testability** is the foundation of TDD. Code must be designed to be easily testable from the start. The key technique is **Dependency Injection (DI)**.

### Anti-Pattern: Tight Coupling

**DO NOT** write code that directly depends on external systems, global state, or non-deterministic values:

```python
# ❌ BAD: Direct dependency on database and current time
def process_order(order_id):
    db = get_database_connection()  # Hard to test
    order = db.get_order(order_id)
    order.processed_at = datetime.now()  # Non-deterministic
    db.save(order)
```

```typescript
# ❌ BAD: Direct dependency on fetch API and Date
async function createOrder(orderData: OrderData) {
  const response = await fetch('/api/orders', {
    method: 'POST',
    body: JSON.stringify(orderData),
  });
  const order = await response.json();
  order.createdAt = new Date().toISOString();
  return order;
}
```

### Correct Pattern: Dependency Injection

**DO** inject dependencies as parameters, making them easy to mock in tests:

```python
# ✅ GOOD: Dependencies injected as parameters
def process_order(order_id, db_session, current_time):
    order = db_session.get_order(order_id)
    order.processed_at = current_time
    db_session.save(order)
    return order

# In production code
process_order(order_id, db_session, datetime.now())

# In test code
mock_db = Mock()
mock_time = datetime(2024, 1, 1, 12, 0, 0)
process_order(123, mock_db, mock_time)
```

```typescript
# ✅ GOOD: Dependencies injected as parameters
async function createOrder(
  orderData: OrderData,
  apiClient: ApiClient,
  dateProvider: () => string
): Promise<Order> {
  const order = await apiClient.post('/api/orders', orderData);
  order.createdAt = dateProvider();
  return order;
}

# In production code
createOrder(data, fetchApiClient, () => new Date().toISOString());

# In test code
const mockApiClient = { post: vi.fn().mockResolvedValue({ id: 1 }) };
const mockDate = '2024-01-01T12:00:00Z';
createOrder(data, mockApiClient, () => mockDate);
```

### Red Flag: "Testing is Hard"

If writing tests feels difficult or requires complex setup, it's a **code smell**. The code is likely:
- Too tightly coupled to external dependencies
- Using global state
- Having unclear responsibilities
- Missing proper abstraction layers

**Solution:** Refactor to improve testability through dependency injection and better separation of concerns.
