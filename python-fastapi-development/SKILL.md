---
name: python-fastapi-development
description: >-
  Python FastAPI backend development workflow covering project setup, database
  integration with SQLAlchemy 2.0, Pydantic validation, authentication,
  error handling, testing, and deployment. Use when building FastAPI APIs,
  setting up new Python backends, or following a structured development workflow.
---

# Python FastAPI Development Workflow

## Overview

Structured workflow for building production-ready Python backends with FastAPI, SQLAlchemy ORM, Pydantic validation, and clean architecture patterns.

## Technology Stack

| Category    | Technology           |
|-------------|----------------------|
| Framework   | FastAPI              |
| Language    | Python 3.11+         |
| ORM         | SQLAlchemy 2.0       |
| Validation  | Pydantic v2          |
| Database    | PostgreSQL           |
| Migrations  | Alembic              |
| Auth        | JWT (python-jose)    |
| Testing     | pytest               |
| Package Mgr | uv or pip            |

## Phase 1: Project Setup

```bash
# Initialize with uv
uv init myproject && cd myproject
uv add fastapi uvicorn[standard] sqlalchemy psycopg2-binary pydantic-settings
uv add alembic python-jose[cryptography] passlib[bcrypt] python-dotenv

# Create structure
mkdir -p backend/{routes,controllers,repositories,utils,validation,models,core,middlewares}
mkdir -p common/{models,database}
mkdir -p database/{alembic/versions,schemas,seeders}
```

### App Entry Point

```python
# backend/app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from backend.routes.router import router
from backend.core.config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: load caches, warm connections
    yield
    # Shutdown: cleanup resources

app = FastAPI(title="My API", lifespan=lifespan)
app.add_middleware(CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"])
app.include_router(router)
```

## Phase 2: Database Setup

### SQLAlchemy Models (in `common/models/`)

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, DateTime, func

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

### Database Session (in `common/database/`)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from backend.core.config import settings

engine = create_engine(settings.DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

def get_db_session():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Alembic Setup

```bash
cd database && alembic init alembic
# Edit alembic/env.py to import common.base.Base and common.models
```

## Phase 3: API Layers

### Routes (declarations only)

```python
# backend/routes/users.py
from fastapi import APIRouter
from backend.controllers.users import get_users, create_user
from backend.validation.users import UserCreate, UserResponse

router = APIRouter(prefix="/api/users", tags=["Users"])
router.add_api_route("/", get_users, methods=["GET"],
                     response_model=list[UserResponse])
router.add_api_route("/", create_user, methods=["POST"],
                     response_model=UserResponse, status_code=201)
```

### Controllers (orchestration, no DB)

```python
# backend/controllers/users.py
from fastapi import Depends, HTTPException
from sqlalchemy.orm import Session
from backend.core.dependencies import get_db_session
from backend.repositories.user_repository import UserRepository
from backend.validation.users import UserCreate, UserResponse

def get_users(db: Session = Depends(get_db_session)):
    repo = UserRepository(db)
    return [UserResponse.model_validate(u) for u in repo.get_all()]

def create_user(data: UserCreate, db: Session = Depends(get_db_session)):
    repo = UserRepository(db)
    if repo.get_by_email(data.email):
        raise HTTPException(400, "Email already registered")
    user = repo.create(data)
    return UserResponse.model_validate(user)
```

### Repositories (all DB access)

```python
# backend/repositories/user_repository.py
from sqlalchemy.orm import Session
from common.models.user import User

class UserRepository:
    def __init__(self, db: Session):
        self._db = db

    def get_all(self) -> list[User]:
        return self._db.query(User).all()

    def get_by_email(self, email: str) -> User | None:
        return self._db.query(User).filter(User.email == email).first()

    def create(self, data) -> User:
        user = User(email=data.email, password_hash=hash_password(data.password))
        self._db.add(user)
        self._db.commit()
        self._db.refresh(user)
        return user
```

### Validation (Pydantic schemas)

```python
# backend/validation/users.py
from pydantic import BaseModel, EmailStr, Field, ConfigDict

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: EmailStr
    created_at: datetime
```

## Phase 4: Authentication

```python
# backend/core/dependencies.py
from fastapi import Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from backend.utils.auth_utils import decode_jwt

security = HTTPBearer()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db_session),
):
    payload = decode_jwt(credentials.credentials)
    repo = UserRepository(db)
    user = repo.get_by_id(payload["sub"])
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

# backend/utils/auth_utils.py
from jose import jwt, JWTError
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"])

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_jwt(data: dict) -> str:
    return jwt.encode(data, settings.SECRET_KEY, algorithm="HS256")

def decode_jwt(token: str) -> dict:
    try:
        return jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
    except JWTError:
        raise HTTPException(401, "Invalid token")
```

## Phase 5: Error Handling

```python
# backend/core/exceptions.py
class AppError(Exception):
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code

class NotFoundError(AppError):
    def __init__(self, resource: str):
        super().__init__(f"{resource} not found", 404)

# Register in app.py
@app.exception_handler(AppError)
async def app_error_handler(request, exc: AppError):
    return JSONResponse(status_code=exc.status_code,
                        content={"error": exc.message})
```

## Phase 6: Testing

```python
# backend/test/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from common.base import Base

@pytest.fixture
def db_session():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()

# backend/test/test_users.py
def test_create_user(db_session):
    repo = UserRepository(db_session)
    user = repo.create(UserCreate(email="test@test.com", password="12345678"))
    assert user.email == "test@test.com"
    assert repo.get_by_email("test@test.com") is not None
```

## Phase 7: Configuration

```python
# backend/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    DATABASE_URL: str
    SECRET_KEY: str
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    API_HOST: str = "0.0.0.0"
    API_PORT: int = 9015

settings = Settings()
```

## Quality Gates

- [ ] All tests passing
- [ ] Type checking passes (mypy or pyright)
- [ ] Linting clean (ruff or flake8)
- [ ] No DB access outside repositories
- [ ] Controllers are pure functions (no decorators)
- [ ] Routes only contain declarations
- [ ] API documentation auto-generated via OpenAPI
- [ ] Migrations have both upgrade and downgrade
