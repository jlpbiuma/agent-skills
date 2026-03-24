# Pydantic v2 — Advanced Reference

## Built-in Field Types

```python
from pydantic import (
    BaseModel, EmailStr, HttpUrl, UUID4,
    FilePath, DirectoryPath, Json, SecretStr,
    PositiveInt, NegativeFloat, conint, constr
)
from typing import Literal

class Example(BaseModel):
    email: EmailStr
    website: HttpUrl
    id: UUID4
    config_file: FilePath
    data_dir: DirectoryPath
    metadata: Json[dict[str, str]]
    api_key: SecretStr
    age: PositiveInt
    balance: NegativeFloat
    username: constr(min_length=3, max_length=20, pattern=r'^[a-z]+$')
    code: conint(ge=1000, le=9999)
    status: Literal['pending', 'approved', 'rejected']
```

## Custom Types

```python
from pydantic import GetCoreSchemaHandler
from pydantic_core import core_schema
from typing import Any

class Color:
    def __init__(self, r: int, g: int, b: int):
        self.r, self.g, self.b = r, g, b

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.no_info_after_validator_function(
            cls.validate, core_schema.str_schema()
        )

    @classmethod
    def validate(cls, v: str) -> 'Color':
        if not v.startswith('#') or len(v) != 7:
            raise ValueError('Invalid hex color')
        return cls(int(v[1:3], 16), int(v[3:5], 16), int(v[5:7], 16))
```

## Type Coercion and Strict Mode

```python
from pydantic import BaseModel, ConfigDict
from typing import Annotated

# Strict mode — no coercion
class StrictModel(BaseModel):
    model_config = ConfigDict(strict=True)
    count: int    # "42" would raise ValidationError

# Per-field strict
class MixedModel(BaseModel):
    flexible: int                              # Allows coercion
    strict: Annotated[int, Field(strict=True)] # No coercion
```

## Nested Models and Recursive Types

```python
class Address(BaseModel):
    street: str
    city: str

class Company(BaseModel):
    name: str
    address: Address

# Recursive (tree)
class TreeNode(BaseModel):
    value: int
    children: list['TreeNode'] = []

TreeNode.model_rebuild()
```

## Generic Models

```python
from typing import Generic, TypeVar

T = TypeVar('T')

class Response(BaseModel, Generic[T]):
    success: bool
    data: T
    message: str = ''

user_response = Response[User](success=True, data=User(id=1, name='Alice'))
list_response = Response[list[User]](success=True, data=[...])
```

## Custom Serializers

```python
from pydantic import model_serializer

class User(BaseModel):
    id: int
    username: str
    password: SecretStr

    @model_serializer
    def ser_model(self) -> dict[str, Any]:
        return {'id': self.id, 'username': self.username}
        # password excluded from serialization
```

## Multi-Environment Settings

```python
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    environment: Literal['dev', 'staging', 'prod'] = 'dev'
    database_url: str

    @property
    def is_production(self) -> bool:
        return self.environment == 'prod'

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

## Query Parameters (FastAPI)

```python
from pydantic import BaseModel, Field
from typing import Literal

class SearchParams(BaseModel):
    q: str = Field(..., min_length=1)
    category: str | None = None
    sort_by: Literal['date', 'relevance'] = 'relevance'

@app.get('/search')
def search(params: SearchParams = Query()):
    return {'query': params.q}
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

    @computed_field
    @property
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

rect = Rectangle(width=10, height=5)
data = rect.model_dump()
# {'width': 10.0, 'height': 5.0, 'area': 50.0, 'perimeter': 30.0}
```

## Custom Errors

```python
from pydantic_core import PydanticCustomError

class StrictUser(BaseModel):
    username: str

    @field_validator('username')
    @classmethod
    def validate_username(cls, v: str) -> str:
        if len(v) < 3:
            raise PydanticCustomError(
                'username_too_short',
                'Username must be at least 3 characters',
                {'min_length': 3, 'actual_length': len(v)}
            )
        return v
```

## Performance Optimization

```python
from pydantic import BaseModel, ConfigDict

class OptimizedModel(BaseModel):
    model_config = ConfigDict(
        validate_assignment=False,
        validate_default=False,
    )
    data: list[int]

# Bulk validation
def validate_bulk(items: list[dict]) -> list[MyModel]:
    return [MyModel.model_validate(item) for item in items]
```

## JSON Schema Generation

```python
class Product(BaseModel):
    id: int = Field(description="Unique identifier")
    name: str = Field(description="Product name", examples=["Widget"])
    price: float = Field(gt=0, description="Price in USD")

schema = Product.model_json_schema()
# Generates OpenAPI-compatible JSON Schema
```

## Dataclass Integration

```python
from pydantic.dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str = Field(min_length=1)
    email: str = Field(pattern=r'.+@.+\..+')

user = User(id=1, name='Alice', email='alice@example.com')
```

## Testing Strategies

```python
import pytest
from pydantic import ValidationError

def test_valid_data():
    user = User(id=1, name='Alice', email='alice@example.com')
    assert user.name == 'Alice'

def test_invalid_data():
    with pytest.raises(ValidationError) as exc_info:
        User(id='invalid', name='Bob', email='bob@example.com')
    assert exc_info.value.errors()[0]['type'] == 'int_parsing'

def test_serialization():
    user = User(id=1, name='Alice', email='alice@example.com')
    data = user.model_dump()
    assert data == {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'}

# Property-based testing
from hypothesis import given, strategies as st

@given(id=st.integers(min_value=1), name=st.text(min_size=1, max_size=100))
def test_always_valid(id, name):
    user = User(id=id, name=name, email='test@test.com')
    assert user.id == id
```

## Full v1 → v2 Migration Guide

### Method Changes

| v1                     | v2                              |
|------------------------|---------------------------------|
| `class Config:`        | `model_config = ConfigDict()`   |
| `.dict()`              | `.model_dump()`                 |
| `.json()`              | `.model_dump_json()`            |
| `.parse_obj(data)`     | `.model_validate(data)`         |
| `.parse_raw(json_str)` | `.model_validate_json(json_str)`|
| `@validator`           | `@field_validator` + `@classmethod` |
| `@root_validator`      | `@model_validator(mode='after')`|
| `json_encoders`        | `@field_serializer`             |

### Migration Checklist

- [ ] Replace `class Config` with `model_config = ConfigDict()`
- [ ] Update `.dict()` → `.model_dump()`
- [ ] Update `.json()` → `.model_dump_json()`
- [ ] Update `.parse_obj()` → `.model_validate()`
- [ ] Update `@validator` → `@field_validator` with `@classmethod`
- [ ] Update `@root_validator` → `@model_validator(mode='after')`
- [ ] Replace `json_encoders` → `@field_serializer`
- [ ] Update custom types to use `__get_pydantic_core_schema__`

## Pagination Pattern

```python
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int

    @computed_field
    @property
    def total_pages(self) -> int:
        return (self.total + self.page_size - 1) // self.page_size
```

## Audit Mixin Pattern

```python
class AuditMixin(BaseModel):
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    created_by: int | None = None

class Document(AuditMixin):
    title: str
    content: str
```

## Resources

- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Migration Guide v1→v2](https://docs.pydantic.dev/latest/migration/)
- [Performance Benchmarks](https://docs.pydantic.dev/latest/concepts/performance/)
