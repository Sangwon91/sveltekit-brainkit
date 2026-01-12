# Testing Strategy (TDD)

**"Test First" is not a suggestion, it is the rule.**

## 1. The TDD Cycle

1.  **Red**: Write a failing test for the *Public Interface* of a slice.
2.  **Green**: Write the minimal code in `packages/core` to pass the test.
3.  **Refactor**: Clean up the code while keeping tests green.

## 2. Testing Layers

### Unit Tests (`packages/core/tests`)
-   **Target**: `service.py`, domain logic.
-   **Speed**: Instant.
-   **Mocks**: Heavy use of Mocks/Fakes for external I/O (Database, APIs).
-   **Philosophy**: Test *logic branches*, not just happy paths.

### Integration Tests (`apps/api/tests`)
-   **Target**: FastAPI Endpoints (`GET /orders`, `POST /login`).
-   **Speed**: Slower (spins up TestClient).
-   **Philosophy**: Verify that the API correctly *wires up* the Core Logic and handles HTTP concerns (status codes, serialization).
