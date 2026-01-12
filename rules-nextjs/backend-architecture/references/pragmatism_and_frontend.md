# Pragmatism & Frontend Integration

## 1. Pragmatic Implementation Details

### DTOs
-   Use **Pydantic V2**.

### Protocol vs ABC
Prefer **Protocols** (Implicit Interfaces) over Abstract Base Classes (Explicit Inheritance) to decouple components.

#### ✅ GOOD: Protocol (Implicit)
```python
class Repository(Protocol):
    def save(self, item: Item) -> None: ...
```

#### ❌ BAD: ABC (Explicit Inheritance coupling)
```python
class BaseRepository(ABC):
    @abstractmethod
    def save(self, item: Item) -> None: ...
```

### Avoid Over-Engineering
-   Do not create a `UseCases` folder if a simple function in `service.py` suffices.
-   Do not create `IUnitOfWork` if a simple context manager works.

## 2. Frontend Integration

-   **OpenAPI Generator**: The frontend uses `openapi-typescript-codegen`.

### Requirements
-   Ensure `operation_id` in FastAPI routes is unique and meaningful (e.g., `orders_get_by_id`, not `read_item`).
-   Use proper Pydantic models for request/response bodies to ensure generated TypeScript types are accurate.
