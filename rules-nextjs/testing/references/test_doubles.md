# Rule 3: Test Doubles (Fake, Stub, Mock, Spy)

## Understanding Test Doubles

When testing, you often need to replace real dependencies with **test doubles**. Understanding the different types helps you choose the right tool:

### Fake

A **Fake** is a working implementation that takes a shortcut (e.g., in-memory database). It is the **preferred** test double in Chicago School TDD because it maintains state and contract.

```python
# ✅ Preferred: Fake Repository
class FakeUserRepository:
    def __init__(self):
        self.users = {}
    
    def save(self, user):
        self.users[user.id] = user
        
    def get(self, user_id):
        return self.users.get(user_id)
```

### Stub

A **Stub** provides predefined responses to method calls. It doesn't verify how it's called. Use it when you just need to provide data to the system under test.

```python
# Stub: Returns fixed value
mocker.patch('module.get_discount', return_value=0.1)
```

### Mock

A **Mock** verifies that specific methods were called with expected arguments. Use it when interaction (protocol) verification is critical.

```python
# Mock: Verify interaction
mock_service.send.assert_called_once_with(...)
```

### Spy

A **Spy** wraps a real object and records method calls. Use it when you want to use the real object but also verify how it was used.

```python
# Spy: Verify call on real object
spy_logger = mocker.spy(real_logger, 'log')
process(logger=real_logger)
spy_logger.assert_called()
```

### When to Use Test Doubles: Pragmatic Guidelines

**⚠️ Critical Principle:** **Use real objects first**. Fake and Mock should only be used when absolutely necessary. Test doubles should be used **sparingly**. Over-mocking makes tests brittle and reduces refactoring safety.

#### Use Real Objects When Possible

**DO** use real objects for:
- **Fast, safe dependencies:** Domain models, value objects, pure functions
- **In-memory operations:** Collections, data structures
- **Any dependency that can be used directly without side effects or performance issues**

**Remember**: Even if you can create a Fake or Mock, prefer the real object unless there's a compelling reason not to (e.g., slow external API, uncontrolled side effects, expensive operations).

#### Use Fakes/Stubs When Necessary

**DO** use **Fake Objects** (preferred) or Stubs for:
- **Slow dependencies:** External APIs, databases
- **External services:** Payment gateways, email services (use Fake to store state)

#### Avoid Protocol Verification Obsession

**DO NOT** obsess over verifying every method call. If protocol verification becomes too complex, switch to **state verification** using Real Objects or Fakes.
