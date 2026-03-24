---
name: architecture-patterns
description: >-
  Enforce clean backend architecture with strict layered boundaries: Repository
  Pattern, Pure Entities, Thin Controllers, Explicit Routers, and a shared
  internal package. Use when writing new endpoints, creating modules, enforcing
  dependency boundaries, or refactoring backend code.
---

# Backend Architecture Patterns

Strict layered architecture ensuring each layer has a single responsibility and dependencies flow in one direction.

## When to Use This Skill

- Writing new endpoints or backend modules
- Enforcing dependency boundaries between layers
- Reviewing code for architectural violations
- Implementing the Repository or Entity pattern

## Core Pillars

### 1. Internal Package (`common/`)

Single source of truth for the database schema.

**Rule**: `common/models` defines SQLAlchemy columns only. No business logic, no framework dependencies.

### 2. Repository Pattern (`repositories/`)

The ONLY layer allowed to touch the database.

**Rule**: All `Select`, `Insert`, `Update` operations happen here. Controllers and utils are forbidden from importing `Session` or executing queries directly.

### 3. Pure Entities (`entities/`)

Rich models containing pure business logic (e.g., `.is_valid()`, `.calculate_total()`).

**Rule**: Entities MUST NOT receive `Session` objects. They operate entirely in memory, oblivious to infrastructure.

### 4. Domain Services (`domain/`) — Optional

Orchestration logic spanning multiple entities or aggregates.

**Rule**: Only create domain services when crossing aggregate boundaries. Simple CRUD does not need this layer.

### 5. Thin Controllers + Explicit Routers

Separation of HTTP metadata from business logic.

**Rule**: `routes/` handle `add_api_route()` and Pydantic schemas. `controllers/` are pure Python functions with NO decorators, making them testable without the framework.

### 6. Validation Schemas (`validation/` or `schemas/`)

Pydantic models for API request/response contracts.

**Rule**: Validation schemas live in their own directory, separate from ORM models. Never mix SQLAlchemy and Pydantic in the same file.

## Implementation Example

```python
# routes/product.py — HTTP metadata only
from fastapi import APIRouter, Depends
from controllers.product import search_products

router = APIRouter()
router.add_api_route("/products/search", search_products,
                     methods=["GET"], response_model=list[ProductResponse])

# controllers/product.py — NO decorators, pure Python
def search_products(q: str, db=Depends(get_session)):
    repo = ProductRepository(db)
    return repo.search(q)

# repositories/product.py — ONLY layer touching DB
class ProductRepository:
    def __init__(self, session):
        self._session = session

    def search(self, query: str):
        return self._session.query(Product).filter(
            Product.name.ilike(f"%{query}%")
        ).all()

# entities/product.py — pure business logic, no DB
class ProductEntity:
    def __init__(self, price: float, stock: int):
        self.price = price
        self.stock = stock

    def is_available(self) -> bool:
        return self.stock > 0

    def apply_discount(self, percentage: float) -> float:
        return self.price * (1 - percentage / 100)
```

## Dependency Flow

```
routes/ → controllers/ → repositories/ → common/models/
                       → entities/      (pure, no DB)
                       → domain/        (orchestration)
```

**Allowed imports per layer:**

| Layer          | Can Import                                  | Cannot Import              |
|----------------|---------------------------------------------|----------------------------|
| Routes         | Controllers, Validation schemas             | Repositories, DB Session   |
| Controllers    | Repositories, Entities, Domain, Validation  | DB Session directly        |
| Repositories   | common/models, DB Session                   | Controllers, Routes        |
| Entities       | Nothing (pure Python)                       | Session, Repositories      |
| Domain         | Repositories, Entities                      | Controllers, Routes        |

## Violation Flags

- **Controller DB Access**: A controller executing `session.execute(select(Product))` — move to repository
- **Impure Entity**: An entity receiving `Session` or making HTTP requests — keep entities pure
- **Fat Router**: A router containing `if/else` logic or data transformations — move to controller
- **Common Contamination**: `common/models/` importing framework code or containing business methods — keep it schema-only
- **Raw SQL in Controllers**: Using `text()` queries directly — use ORM via repositories

## Good Practices

```python
# GOOD: Controller delegates to repository
def get_user(user_id: int, db=Depends(get_session)):
    repo = UserRepository(db)
    user = repo.get_by_id(user_id)
    if not user:
        raise HTTPException(404)
    return UserResponse.model_validate(user)

# GOOD: Entity is pure Python
class OrderEntity:
    def calculate_total(self, items: list[Item]) -> float:
        return sum(item.price * item.quantity for item in items)
```

## Bad Practices

```python
# BAD: Controller touches DB directly
def get_user(user_id: int, db=Depends(get_session)):
    return db.query(User).filter(User.id == user_id).first()

# BAD: Entity receives Session
class OrderEntity:
    def save(self, session: Session):
        session.add(self)

# BAD: Router contains business logic
@router.post("/orders")
async def create_order(data: OrderCreate, db=Depends(get_session)):
    if data.total > 1000:  # Logic in router!
        data.discount = 10
    db.add(Order(**data.dict()))
```

## Checklist

- [ ] `common/models/` contains only SQLAlchemy column definitions
- [ ] All DB queries live in `repositories/`
- [ ] Controllers are pure functions without decorators
- [ ] Routes only declare endpoints via `add_api_route()`
- [ ] Entities have no infrastructure dependencies
- [ ] Pydantic schemas live in `validation/` or `schemas/`, not `models/`
- [ ] Domain services only exist for cross-aggregate orchestration
