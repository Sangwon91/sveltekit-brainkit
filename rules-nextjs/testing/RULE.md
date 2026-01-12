---
alwaysApply:  true
---

# Testing Rules (TDD & Strategy)

## Philosophy
**Test-First Development** is mandatory, not optional. We adhere to the **Red-Green-Refactor** cycle to drive design and quality. Code must be designed for **Testability** via Dependency Injection from day one. We prefer state verification (Chicago School) and use Test Doubles pragmatically (prefer Fakes over Mocks). We favor **functional architecture** to enhance testability through pure functions, immutability, and clear separation of concerns.

## MANDATORY TDD WORKFLOW

**CRITICAL: You MUST follow this exact sequence for EVERY feature or function. Skipping steps or combining steps is STRICTLY FORBIDDEN.**

### Step 1: 🔴 RED - Write Failing Test First

**BEFORE writing any implementation code:**

1. Write a test that defines the desired behavior
2. **MUST run the test** to confirm it fails (Red state)
3. **MUST verify the failure** - if the test does NOT fail, you have written the test incorrectly
4. **DO NOT proceed** to Step 2 until you have confirmed the test fails

**What to write:**
- Test file (e.g., `feature.test.ts`, `test_feature.py`)
- Test cases that describe expected behavior
- Assertions that verify the behavior

**What NOT to write:**
- ❌ Implementation code
- ❌ Production code files
- ❌ Any code that makes the test pass

**Verification command examples:**
- Python: `pytest path/to/test_file.py -v`
- Next.js: `npm test path/to/test_file.test.ts` or `vitest run path/to/test_file.test.ts`

### Step 2: 🟢 GREEN - Make Test Pass with Minimal Code

**ONLY AFTER Step 1 is complete and test failure is confirmed:**

1. Write the **minimum code necessary** to make the test pass
2. **MUST run the test** to confirm it passes (Green state)
3. **MUST verify the pass** - if the test does NOT pass, fix the implementation
4. **DO NOT proceed** to Step 3 until you have confirmed the test passes

**What to write:**
- Minimal implementation (hardcoding is acceptable)
- Only the code needed to satisfy the test
- Simple, straightforward solution

**What NOT to write:**
- ❌ Optimized code
- ❌ Refactored code
- ❌ Additional features beyond what the test requires

**Verification command examples:**
- Python: `pytest path/to/test_file.py -v`
- Next.js: `npm test path/to/test_file.test.ts` or `vitest run path/to/test_file.test.ts`

### Step 3: 🔵 REFACTOR - Improve Code Quality

**ONLY AFTER Step 2 is complete and test pass is confirmed:**

1. Improve code structure, naming, and organization
2. Remove duplication and improve readability
3. **MUST run the test** after each refactoring change
4. **MUST verify the test still passes** - if it fails, revert and try a different approach
5. **DO NOT change behavior** - only improve code quality

**What to do:**
- Extract functions/methods
- Improve variable names
- Remove code duplication
- Optimize performance (without changing behavior)

**What NOT to do:**
- ❌ Change test cases (unless fixing a bug in the test itself)
- ❌ Add new features (write a new test first)
- ❌ Change behavior or expected outcomes

### ABSOLUTE PROHIBITIONS

**You MUST NEVER:**

1. ❌ Write test and implementation code in the same step
2. ❌ Skip running tests between steps
3. ❌ Proceed to the next step without verifying the current step's result
4. ❌ Combine Red, Green, or Refactor phases
5. ❌ Write implementation code before writing and running a failing test
6. ❌ Refactor before confirming tests pass
7. ❌ Skip test execution and assume results
8. ❌ Create branching points in source code for testing environments (e.g., `if (process.env.NODE_ENV === 'test')`, `if (isTestMode)`). This introduces additional errors and increases code complexity. Use Dependency Injection and Test Doubles instead.

### Workflow Summary

```
For EACH feature/function:
  1. Write test → Run test → Verify FAILURE (Red) ✅
  2. Write minimal code → Run test → Verify PASS (Green) ✅
  3. Refactor code → Run test → Verify PASS (Green) ✅
  4. Move to next feature/function
```

**If you are asked to implement a feature:**
1. Start with Step 1 (Red) - write and run failing test
2. Show the test failure output
3. Then proceed to Step 2 (Green) - write minimal code
4. Show the test pass output
5. Then proceed to Step 3 (Refactor) - improve code
6. Show the test still passes after refactoring

## Core Knowledge & Mini-Examples

### 1. The TDD Cycle
- **Red**: Write a failing test first. Define instructions and expect behavior.
- **Green**: Write minimal code to pass the test. Hardcoding is allowed.
- **Refactor**: Clean up implementation while keeping tests green.

### 2. Dependency Injection (DI)
- **Do**: Inject dependencies (API clients, DB) as arguments/props.
- **Don't**: Import valid instances or globals inside functions.

```python
# ✅ Good: Injectable
def process_order(order, db): ...

# ❌ Bad: Hard dependency
def process_order(order):
    db = get_db() ...
```

### 3. Test Doubles Strategy
- **Real Objects First**: Use real objects whenever possible. Fake and Mock should only be used when absolutely necessary (e.g., slow external dependencies, side effects that cannot be controlled).
- **Fake**: Working implementation (e.g., in-memory DB). **Preferred** for external services when real objects cannot be used.
- **Stub**: Canned answers. Use for providing data.
- **Mock**: Expectation verification. Use sparingly and only when interaction verification is critical.
- **Spy**: Wrap real object to inspect calls.

### 4. Unit Test Four Pillars
When writing unit tests, consider these four essential qualities. If it's difficult to satisfy all of them, prioritize **Refactoring Safety** as the most important:

1. **Regression Prevention**: Tests should catch bugs when behavior changes unexpectedly.
2. **Refactoring Safety** (Most Important): Tests should remain stable when implementation changes but behavior remains the same. Avoid over-specification and coupling to implementation details.
3. **Fast Feedback**: Tests should run quickly to provide immediate feedback during development.
4. **Maintainability**: Tests should be easy to read, understand, and modify. Clear test names and structure are essential.

**Priority Order**: Refactoring Safety > Regression Prevention > Fast Feedback > Maintainability

### 5. Schools of TDD
- **Chicago (Classical)**: Verify State. Real objects or Fakes. **Default & Preferred**.
- **London (Mockist)**: Verify Behavior/Interaction. Use Mocks. Use only for side-effects.

## Tech Stack
- **Python**: `pytest`
- **Next.js**: `Vitest` or `Jest`, `React Testing Library`, `MSW` (for API integration)

## Workflow
1.  **Write Failing Test**: Define the requirement in a new test file.
2.  **Run Test**: Confirm it fails (Red).
3.  **Implement**: Write minimal code to satisfy the test.
4.  **Verify**: Confirm it passes (Green).
5.  **Refactor**: Improve code structure without changing behavior.
6.  **Repeat**: Move to the next requirement.

## References
- **TDD Cycle**: See [Red-Green-Refactor Process](references/tdd_cycle.md)
- **Design for Testing**: See [Dependency Injection Patterns](references/dependency_injection.md)
- **Doubles Guide**: See [Fakes, Stubs, and Mocks](references/test_doubles.md)
- **Methodology**: See [Chicago vs London School](references/testing_schools.md)
- **Test Scopes**: See [Solitary vs Sociable Tests](references/test_depth.md)
