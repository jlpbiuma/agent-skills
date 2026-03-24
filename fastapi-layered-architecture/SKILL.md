---
name: fastapi-layered-architecture
description: >-
  Design FastAPI applications with clean layered architecture: thin routes,
  controller functions, repositories, utils, ORM models, and Pydantic
  validation. Use when building FastAPI APIs, structuring endpoint logic,
  separating concerns, or reviewing Python API code organization.
---

# FastAPI Layered Architecture

## Layer Stack

```
Request → Route → Controller → Repository / Utils → ORM Model → Database
                                    ↓
Response ← Route ← Controller ← Pydantic Validation ← Data
```

## Directory Structure

```
backend/
├── app.py                    # FastAPI app, lifespan, CORS, middlewares
├── core/
│   ├── config.py             # Settings (Pydantic BaseSettings or env)
│   ├── cors.py               # CORS configuration
│   ├── dependencies.py       # FastAPI Depends: auth, DB sessions
│   └── exceptions.py         # Custom exception classes
├── routes/                   # ONLY route declarations
│   ├── router.py             # Aggregates all route modules
│   └── {domain}.py           # router.add_api_route() calls
├── controllers/              # Endpoint handler functions
│   └── {domain}/             # One module per domain
├── repositories/             # Data access layer (ORM queries)
│   └── {domain}_repository.py
├── utils/                    # Pure transformations, helpers
│   └── {domain}_utils.py
├── validation/               # Pydantic request/response models
│   └── {domain}.py
├── models/                   # SQLAlchemy ORM models (or re-exports)
│   └── {entity}.py
├── middlewares/               # Auth, logging, rate limiting
│   └── auth_middleware.py
└── entities/                 # Domain helpers (optional)
```

## Layer Responsibilities

| Layer            | Files                     | Does                                      | Does NOT                        |
|------------------|---------------------------|-------------------------------------------|---------------------------------|
| **Routes**       | `routes/*.py`             | Declare endpoints (`add_api_route`)       | Define functions, business logic|
| **Controllers**  | `controllers/*/`          | Orchestrate repos + utils, return response| Contain DB queries, helper logic|
| **Repositories** | `repositories/*.py`       | Encapsulate ORM queries                   | Transform data, call APIs       |
| **Utils**        | `utils/*.py`              | Pure transformations, calculations        | Access databases, call repos    |
| **Validation**   | `validation/*.py`         | Define Pydantic request/response schemas  | Contain logic                   |
| **Models (ORM)** | `models/*.py`             | Define SQLAlchemy table mappings          | Contain business logic          |
| **Core**         | `core/*.py`               | Config, deps, exceptions, CORS           | Endpoint-specific logic         |
| **Middlewares**   | `middlewares/*.py`        | Cross-cutting: auth, logging              | Domain-specific logic           |

## Good Practices

### Routes: Only Declarations

```python
# GOOD: routes/recommendations.py
from backend.controllers.recommendations import get_recommendations, get_by_competencies

router = APIRouter(prefix="/api/recommendations", tags=["Recommendations"])

router.add_api_route("/", get_recommendations, methods=["POST"],
                     response_model=RecommendationResponse)
router.add_api_route("/by-competencies", get_by_competencies, methods=["POST"],
                     response_model=RecommendationResponse)
```

### Controllers: Standalone Functions with Depends

```python
# GOOD: controllers/recommendations/get_recommendations.py
from fastapi import Depends
from sqlalchemy.orm import Session
from backend.core.dependencies import get_db_session, get_current_user
from backend.repositories.occupation_repository import OccupationRepository
from backend.utils.recommendation_utils import build_user_vector, calculate_similarity
from backend.validation.recommendations import RecommendationRequest, RecommendationResponse

def get_recommendations(
    request: RecommendationRequest,
    db: Session = Depends(get_db_session),
    user = Depends(get_current_user),
) -> RecommendationResponse:
    # 1. Extract competencies (utils — pure function)
    words, collocations = extract_competencies(request.cv_text)

    # 2. Build vector (utils — pure function)
    user_vector = build_user_vector(words, collocations)

    # 3. Query occupations (repository — DB access)
    occupations = OccupationRepository.get_all(db)

    # 4. Calculate similarity (utils — pure function)
    results = calculate_similarity(user_vector, occupations)

    # 5. Return validated response
    return RecommendationResponse(matches=results[:request.top_n])
```

### Repositories: Encapsulated ORM Queries

```python
# GOOD: repositories/occupation_repository.py
from sqlalchemy.orm import Session
from backend.models.occupation import Occupation

class OccupationRepository:
    @staticmethod
    def get_all(db: Session) -> list[Occupation]:
        return db.query(Occupation).all()

    @staticmethod
    def get_by_id(db: Session, occupation_id: int) -> Occupation | None:
        return db.query(Occupation).filter(
            Occupation.id == occupation_id
        ).first()

    @staticmethod
    def search(db: Session, query: str, limit: int = 20) -> list[Occupation]:
        return db.query(Occupation).filter(
            Occupation.name.ilike(f"%{query}%")
        ).limit(limit).all()
```

### Utils: Pure Functions

```python
# GOOD: utils/recommendation_utils.py
import numpy as np

def build_user_vector(words: set[int], collocations: set[int],
                      total_dimension: int) -> np.ndarray:
    vector = np.zeros(total_dimension)
    for word_id in words:
        vector[word_id] = 1.0
    return vector

def calculate_employability(matched: int, total: int) -> float:
    return (matched / total) * 100 if total > 0 else 0.0
```

### Validation: Pydantic Schemas

```python
# GOOD: validation/recommendations.py
from pydantic import BaseModel, Field

class RecommendationRequest(BaseModel):
    cv_text: str = Field(..., min_length=50, description="CV text content")
    top_n: int = Field(10, ge=1, le=100)
    vector_type: str = Field("binary", pattern="^(binary|weighted)$")

class RecommendationResponse(BaseModel):
    matches: list[OccupationMatch]
    total_competencies: int
    employability_score: float
```

## Bad Practices

### Routes with Inline Logic

```python
# BAD: Business logic in routes
@router.post("/recommendations")
async def get_recommendations(request: RecommendationRequest,
                               db: Session = Depends(get_db)):
    words = extract_words(request.cv_text)       # logic in route!
    occupations = db.query(Occupation).all()      # DB query in route!
    results = calculate_similarity(words, occupations)
    return {"matches": results}
```

### Classes in Controllers

```python
# BAD: Class-based controllers
class RecommendationController:
    def __init__(self):
        self.cache = {}

    def get_recommendations(self, request, db):
        # Mixes state, orchestration, and helpers
        pass
```

### DB Queries in Utils

```python
# BAD: Database access in utils
def get_occupation_details(db: Session, occupation_id: int):
    return db.query(Occupation).filter(  # DB access belongs in repository
        Occupation.id == occupation_id
    ).first()
```

### Raw SQL in Controllers

```python
# BAD: Raw SQL instead of ORM
def get_recommendations(request, db):
    result = db.execute(
        text(f"SELECT * FROM occupations WHERE id = :id"),  # raw SQL
        {"id": request.occupation_id}
    ).fetchall()
```

### Mixing Pydantic and SQLAlchemy in Same Directory

```python
# BAD: ORM model and Pydantic schema in models/
# models/occupation.py
class Occupation(Base):          # SQLAlchemy ORM
    __tablename__ = "occupations"

class OccupationResponse(BaseModel):  # Pydantic — should be in validation/
    id: int
    name: str
```

## App Entry Point Pattern

```python
# app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load caches, warm connections
    load_static_data()
    yield
    # Shutdown: cleanup

app = FastAPI(lifespan=lifespan)
app.add_middleware(CORSMiddleware, allow_origins=settings.CORS_ORIGINS, ...)
app.add_middleware(AuthMiddleware)
app.include_router(router)  # Aggregated from routes/router.py
```

## Dependency Injection Pattern

```python
# core/dependencies.py
from fastapi import Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

def get_db_session():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db_session),
):
    token = credentials.credentials
    payload = decode_jwt(token)
    user = UserRepository.get_by_id(db, payload["sub"])
    if not user:
        raise HTTPException(status_code=401)
    return user
```

## Checklist

- [ ] Routes contain ONLY `add_api_route()` or `@router.method()` — no logic
- [ ] Controllers are standalone functions, not classes
- [ ] Controllers use `Depends()` for DB sessions and auth
- [ ] Repositories encapsulate all ORM queries — no raw SQL elsewhere
- [ ] Utils are pure functions — no DB access, no side effects
- [ ] Validation models are Pydantic; ORM models are SQLAlchemy — separate dirs
- [ ] Startup data loading happens in `lifespan`, not at import time
- [ ] Middlewares handle cross-cutting concerns (auth, CORS, logging)
