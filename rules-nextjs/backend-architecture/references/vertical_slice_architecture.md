# Vertical Slice Architecture (VSA) Rules

## 1. The Slice (Feature)

A "Slice" in `apps/` groups everything needed for a feature.

-   **Input**: `schema.py` defines what enters the slice.
-   **Orchestration**: `service.py` coordinates the flow. It calls into `packages/` for heavy lifting.
-   **Data**: `models.py` defines the shape of data.

## 2. Apps vs Packages

### Apps (`apps/*`)
-   **Role**: The "Glue". Wiring protocols (FastAPI), managing state, orchestrating calls to libraries.
-   **Contents**: VSA Slices, Routers, Services.

### Packages (`packages/*`)
-   **Role**: The "Engine". Pure, testable logic.
-   **Criteria for Extraction**: Extract code here if it is complex (e.g., proper scientific calculation), reusable across multiple apps, or distinctly separate from the web layer.
-   **Constraint**: No FastAPI dependencies. Pure Python.

## 3. Data Modeling & Database

-   **SQLModel**: Use `SQLModel` for entities if DTO/ORM mapping becomes verbose or complex. It combines Pydantic and SQLAlchemy, reducing boilerplate.
-   **Pragmatism**: Do not force separate Pydantic DTOs for every layer if they are identical. `SQLModel` instances can often serve as both DB models and API responses in simple slices.
