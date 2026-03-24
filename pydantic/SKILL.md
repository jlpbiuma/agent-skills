---
name: pydantic
description: >-
  Pydantic v2 data validation with type hints, BaseModel, Field constraints,
  validators, serialization, settings management, and FastAPI/SQLAlchemy
  integration. Use when creating Pydantic models, validating API data,
  managing configuration, or integrating with ORMs.
---

# Pydantic v2 Validation

## When to Use

- API request/response validation (FastAPI, Django)
- Settings and configuration management (env variables)
- ORM model validation (SQLAlchemy integration)
- Data parsing and serialization (JSON, dict)
- Type-safe data classes with automatic validation

## Quick Start

```python
from pydantic import BaseModel, Field, EmailStr
from datetime import datetime

class User(BaseModel):
    id: int
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    created_at: datetime = Field(default_factory=datetime.now)
    is_active: bool = True

user = User(id=1, name="Alice", email="alice@example.com")
print(user.model_dump())

# Type coercion (default)
user2 = User(id="2", name="Bob", email="bob@example.com")
assert user2.id == 2  # String "2" coerced to int
```

## BaseModel Configuration

```python
from pydantic import BaseModel, ConfigDict

class MyModel(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True,
        use_enum_values=True,
        from_attributes=True,       # Enable ORM mode
        arbitrary_types_allowed=False
    )
```

## Field Constraints

```python
from pydantic import Field
from typing import Annotated

class Item(BaseModel):
    sku: str = Field(pattern=r'^[A-Z]{3}-\d{4}$')
    price: float = Field(gt=0, le=10000)
    stock: int = Field(ge=0, default=0)
    quantity: Annotated[int, Field(ge=1, le=100)]
    description: str = Field(..., description="Item description",
                             examples=["High-quality widget"])
```

## Validators

### Field Validators

```python
from pydantic import field_validator

class Account(BaseModel):
    username: str
    password: str

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError('must be alphanumeric')
        return v

    @field_validator('password')
    @classmethod
    def password_strong(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError('must be at least 8 characters')
        return v
```

### Model Validators

```python
from pydantic import model_validator
from typing import Self

class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode='after')
    def check_dates(self) -> Self:
        if self.end_date < self.start_date:
            raise ValueError('end_date must be after start_date')
        return self
```

## Serialization

```python
from pydantic import field_serializer

class Article(BaseModel):
    title: str
    tags: list[str]

    @field_serializer('tags')
    def serialize_tags(self, tags: list[str]) -> str:
        return ','.join(tags)

article = Article(title='Guide', tags=['python', 'validation'])
article.model_dump()            # dict
article.model_dump_json()       # JSON string
article.model_dump(exclude={'tags'})
article.model_dump(include={'title'})
article.model_dump(exclude_unset=True)
```

## Settings Management

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file='.env', env_prefix='APP_', case_sensitive=False
    )

    database_url: str
    secret_key: SecretStr
    debug: bool = False
    cors_origins: list[str] = ["http://localhost:3000"]

settings = Settings()  # Reads APP_DATABASE_URL, APP_SECRET_KEY, etc.
```

## FastAPI Integration

```python
from fastapi import FastAPI
from pydantic import BaseModel, EmailStr, ConfigDict

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: EmailStr

@app.post('/users', response_model=UserResponse)
def create_user(user: UserCreate):
    return UserResponse(id=1, email=user.email)
```

## SQLAlchemy Integration

```python
# ORM model (SQLAlchemy)
class UserDB(Base):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)

# Pydantic schema (validation)
class UserSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: str

# Convert ORM → Pydantic
user_orm = db.query(UserDB).first()
user_validated = UserSchema.model_validate(user_orm)
```

## Schema Variants (CRUD Pattern)

```python
class UserBase(BaseModel):
    email: EmailStr

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):
    email: EmailStr | None = None
    password: str | None = None

class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)
    id: int
    created_at: datetime
```

## Generic Response Wrapper

```python
from typing import Generic, TypeVar

T = TypeVar('T')

class APIResponse(BaseModel, Generic[T]):
    success: bool
    data: T | None = None
    error: str | None = None

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
```

## Computed Fields

```python
from pydantic import computed_field

class Rectangle(BaseModel):
    width: float
    height: float

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height
```

## Good Practices

```python
# GOOD: Use Field constraints instead of validators for simple rules
class Good(BaseModel):
    age: int = Field(ge=0, le=150)
    email: EmailStr

# GOOD: Separate schemas per use case
class UserCreate(BaseModel): ...    # Request
class UserResponse(BaseModel): ...  # Response
class UserInDB(BaseModel): ...      # Internal

# GOOD: from_attributes for ORM compatibility
class Schema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
```

## Bad Practices

```python
# BAD: Validator for simple constraint (use Field instead)
class Bad(BaseModel):
    age: int
    @field_validator('age')
    @classmethod
    def check_age(cls, v):
        if v < 0: raise ValueError('invalid')
        return v

# BAD: Mixing ORM and Pydantic in same file
# BAD: Using .dict() instead of .model_dump() (v1 syntax)
# BAD: Returning `any` from services — use typed schemas
```

## v1 → v2 Migration Quick Reference

| v1                    | v2                            |
|-----------------------|-------------------------------|
| `class Config:`       | `model_config = ConfigDict()` |
| `.dict()`             | `.model_dump()`               |
| `.json()`             | `.model_dump_json()`          |
| `.parse_obj()`        | `.model_validate()`           |
| `@validator`          | `@field_validator` + `@classmethod` |
| `@root_validator`     | `@model_validator(mode='after')` |

## Additional Reference

For advanced topics (custom types, generic models, strict mode, performance optimization,
JSON schema generation, dataclass integration, testing strategies, full migration guide),
see [reference.md](reference.md).
