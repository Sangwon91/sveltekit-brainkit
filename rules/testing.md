---
trigger: always_on
glob: "*"
description: "Testing Rules - TDD & Strategy for SvelteKit and Python"
---

# Testing Rules (TDD & Strategy)

## Philosophy

**Test-First Development** is mandatory, not optional. We adhere to the **Red-Green-Refactor** cycle to drive design and quality. Code must be designed for **Testability** via Dependency Injection from day one.

**좋은 단위 테스트의 네 가지 핵심 원칙**을 항상 염두에 두고 테스트를 작성하세요:

1. **리팩토링 내성** (최우선): 구현이 바뀌어도 동작이 같으면 테스트는 통과해야 함
2. **회귀 방지**: 동작이 의도치 않게 바뀌면 테스트가 실패해야 함
3. **빠른 피드백**: 테스트가 빠르게 실행되어야 함
4. **유지보수성**: 테스트가 읽기 쉽고 수정하기 쉬워야 함

> [!IMPORTANT]
> 이 네 원칙은 상호 배타적인 면이 있습니다. **리팩토링 내성을 최대한 확보**하면서 다른 요소들을 균형있게 고려하세요.

## MANDATORY TDD WORKFLOW

> [!CAUTION]
> **CRITICAL: You MUST follow this exact sequence for EVERY feature or function. Skipping steps or combining steps is STRICTLY FORBIDDEN.**

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
- SvelteKit: `npm test path/to/test_file.test.ts` or `vitest run path/to/test_file.test.ts`

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
8. ❌ Create branching points in source code for testing environments (e.g., `if (import.meta.env.MODE === 'test')`, `if (isTestMode)`). Use Dependency Injection and Test Doubles instead.

### Workflow Summary

```
For EACH feature/function:
  1. Write test → Run test → Verify FAILURE (Red) ✅
  2. Write minimal code → Run test → Verify PASS (Green) ✅
  3. Refactor code → Run test → Verify PASS (Green) ✅
  4. Move to next feature/function
```

## Core Knowledge & Mini-Examples

### 1. The TDD Cycle
- **Red**: Write a failing test first. Define instructions and expect behavior.
- **Green**: Write minimal code to pass the test. Hardcoding is allowed.
- **Refactor**: Clean up implementation while keeping tests green.

### 2. Dependency Injection (DI)
- **Do**: Inject dependencies (API clients, DB) as arguments/props.
- **Don't**: Import valid instances or globals inside functions.

```typescript
// ✅ Good: Injectable
function processOrder(order: Order, db: Database) { ... }

// ❌ Bad: Hard dependency
function processOrder(order: Order) {
  const db = getDb(); // Hard to test
  ...
}
```

### 3. Test Doubles Strategy (블라디미르 분류법)

복잡한 분류(Fake, Stub, Mock, Spy) 대신 **두 가지**만 기억하세요:

| 분류 | 목적 | 검증 방식 |
|------|------|-----------|
| **Stub** | 테스트 대상에 **입력(데이터) 제공** | 상태(State) 검증 |
| **Mock** | 테스트 대상이 **외부에 출력했는지 확인** | 행위(Behavior) 검증 |

- **Real Objects First**: 가능하면 실제 객체를 사용하세요.
- **Stub**: 들어오는 의존성(데이터 조회). **호출 검증 금지**.
- **Mock**: 나가는 의존성(이메일, 로그, 분석). 외부 부수효과만 검증.

### 4. Unit Test Four Pillars (네 가지 핵심 원칙)

| 우선순위 | 원칙 | 설명 | Trade-off |
|----------|------|------|----------|
| 1 | **리팩토링 내성** | 구현 변경 시 동작 동일하면 테스트 통과 | Mock 과용 → 내성 파괴 |
| 2 | **회귀 방지** | 의도치 않은 동작 변경 시 테스트 실패 | 지나친 단순화 → 버그 미탐지 |
| 3 | **빠른 피드백** | 테스트가 빠르게 실행됨 | E2E만 사용 → 피드백 지연 |
| 4 | **유지보수성** | 테스트가 읽기/수정 용이 | 과도한 DRY → 가독성 저하 |

> [!IMPORTANT]
> **리팩토링 내성은 타협 불가입니다.** 거짓 양성(false positive)을 만드는 테스트는 신뢰를 파괴합니다. Mock을 최소화하고, 상태 검증(State Verification)을 선호하세요.

### 5. Schools of TDD
- **Chicago (Classical)**: Verify State. Real objects or Fakes. **Default & Preferred**.
- **London (Mockist)**: Verify Behavior/Interaction. Use Mocks. Use only for side-effects.

## Tech Stack

| Language | Tools |
|----------|-------|
| **Python** | `pytest` |
| **SvelteKit** | `Vitest`, `@testing-library/svelte`, `MSW` (for API mocking) |

## References

- See [Red-Green-Refactor Process](testing/references/tdd_cycle.md) for detailed TDD examples
- See [Dependency Injection Patterns](testing/references/dependency_injection.md) for testability design
- See [Fakes, Stubs, and Mocks](testing/references/test_doubles.md) for test double selection
- See [Chicago vs London School](testing/references/testing_schools.md) for methodology philosophy
- See [Solitary vs Sociable Tests](testing/references/test_depth.md) for test scope guidance
