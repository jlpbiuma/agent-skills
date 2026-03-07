---
name: architecture-patterns
description: Implement proven backend architecture patterns specifically tailored for the AI-Commerce Platform v2.0. Use when architecting new features, writing code, or refactoring the backend to strictly enforce the 7 Core Pillars.
---

# Architecture Patterns (AI-Commerce v2.0)

Master the strictly decoupled architecture adopted by the AI-Commerce Platform. This skill ensures all agent-generated code or refactors comply with the 7 Core Pillars outlined in our Architecture Decision Record.

## When to Use This Skill

- Writing new endpoints or LangGraph nodes
- Creating new backend modules
- Enforcing correct dependency boundaries (e.g. ensuring Controllers don't touch DBs)
- Implementing the "Rich Model / Pure Entity" pattern

## The 7 Core Pillars

### 1. Internal Package (`common/`)
The single source of truth for the database schema. 
**Rule**: `common/models` must ONLY define SQLAlchemy columns. It must NEVER contain business logic or depend on external APIs/Frameworks.

### 2. Repository Pattern (`backend/repositories/`)
The ONLY layer allowed to touch the database.
**Rule**: All `Select`, `Insert`, `Update` operations MUST happen here. Controllers and Tools are forbidden from importing `Session` or executing queries directly.

### 3. Pure Entities (`backend/entities/`)
Rich models containing pure business logic (e.g., `.is_available()`, `.add_item()`).
**Rule**: Entities MUST NOT receive `Session` objects as arguments. They operate entirely in memory oblivious to infrastructure.

### 4. Selective Domain Services (`backend/domain/`)
Orchestration logic that spans multiple entities (e.g., Checkout involving Cart, Payment, User).
**Rule**: Do not create Domain Services for simple CRUD operations. Use them only when crossing aggregate boundaries.

### 5. Thin Controllers + Explicit Routers
Separation of HTTP metadata from logic.
**Rule**: `routers/` handle `add_api_route` and Pydantic schemas. `controllers/` are pure Python functions with NO decorators, making them easily testable without FastAPI.

### 6. Tools alongside Controllers (`backend/tools/`)
LangGraph tools mirror Controllers.
**Rule**: Tools (`@tool`) must consume `repositories/` or `domain/` directly. Placed at the same hierarchy as controllers. An Agent must never call an HTTP endpoint to resolve domain logic.

### 7. Explicit Agent I/O Contract (`backend/agents/io.py`)
Controllers interact with LangGraph using strict data structures.
**Rule**: Generate `AgentInput` to trigger the graph and receive `AgentOutput`. Do NOT arbitrarily mutate `state.py` from outside the graph execution flow.

## Implementation Example: Thin Controller & Router

```python
# routers/product.py
from fastapi import APIRouter, Depends
from controllers.product import search_products_controller

router = APIRouter()
router.add_api_route(
    "/products/search",
    search_products_controller,
    methods=["GET"],
    response_model=List[ProductResponse]
)

# controllers/product.py
# NO DECORATORS HERE!
def search_products_controller(q: str, session=Depends(get_session)):
    repo = ProductRepository(session)
    return repo.search(q)
```

## Violation Flags (DO NOT WRITE CODE LIKE THIS)

- ❌ A Controller executing `session.execute(select(Product))`
- ❌ An Entity receiving `SessionLocal`
- ❌ A Router containing business logic inside the route handler
- ❌ A LangGraph Node making a `requests.get("http://localhost:8000/api/products")` instead of using a `tool/`
