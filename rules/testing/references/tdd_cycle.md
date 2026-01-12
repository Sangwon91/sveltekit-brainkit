# Rule 1: Red-Green-Refactor Cycle

## The Fundamental TDD Cycle

**Test-Driven Development (TDD)** follows a strict three-phase cycle that must be repeated for every feature:

### Phase 1: 🔴 Red (Write Failing Test)

- **Action:** Write a test case for the functionality you want to implement **before** writing any implementation code.
- **Purpose:** Define the interface, expected behavior, and requirements clearly. The test should fail because the functionality doesn't exist yet.
- **Key Point:** This phase forces you to think about the API design, input/output contracts, and edge cases upfront.

### Phase 2: 🟢 Green (Make Test Pass)

- **Action:** Write the **minimum code necessary** to make the test pass.
- **Purpose:** Get to a working state quickly. Code quality is secondary at this stage.
- **Key Point:** Even hardcoded values or simple implementations are acceptable. The goal is to see the test turn green.

### Phase 3: 🔵 Refactor (Improve Code Quality)

- **Action:** Improve the code structure, remove duplication, improve naming, and optimize while **keeping all tests passing**.
- **Purpose:** Enhance code quality without changing behavior. The test suite provides a safety net.
- **Key Point:** Refactoring is safe because tests will catch any regressions immediately.

### Example: Python

```python
# 🔴 RED: Write failing test first
def test_calculate_total():
    items = [{"price": 10, "quantity": 2}, {"price": 5, "quantity": 3}]
    assert calculate_total(items) == 35

# 🟢 GREEN: Minimal implementation
def calculate_total(items):
    return 35  # Hardcoded to pass test

# 🔵 REFACTOR: Proper implementation
def calculate_total(items):
    return sum(item["price"] * item["quantity"] for item in items)
```

### Example: SvelteKit (TypeScript)

```typescript
// 🔴 RED: Write failing test first
describe('formatCurrency', () => {
  it('should format number as USD currency', () => {
    expect(formatCurrency(1234.56)).toBe('$1,234.56');
  });
});

// 🟢 GREEN: Minimal implementation
function formatCurrency(amount: number): string {
  return '$1,234.56'; // Hardcoded
}

// 🔵 REFACTOR: Proper implementation
function formatCurrency(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(amount);
}
```
