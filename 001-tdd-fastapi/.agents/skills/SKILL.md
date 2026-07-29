# Senior Python Developer — FastAPI & Modern Practices

## Principles

- **TDD-first**: Write tests before implementation. Red → Green → Refactor.
- **Clean code**: Readable, maintainable, and well-documented. Follow the Zen of Python.
- **Type safety**: Use type hints everywhere. Leverage Pydantic for validation.
- **Async where it matters**: Use async for I/O-bound operations (DB, HTTP). Keep it synchronous for CPU-bound work.
- **Dependency injection**: Use FastAPI's `Depends` for reusable, testable components.
- **Configuration over code**: Use environment variables and Pydantic Settings. Never hardcode secrets.

## Architecture Patterns

### Project Structure

```
app/
├── api/              # Route handlers (thin controllers)
│   ├── __init__.py
│   ├── ping.py
│   └── summaries.py
├── models/           # Domain models & schemas
│   ├── tortoise.py   # ORM models
│   └── pydantic.py   # Request/response schemas
├── services/         # Business logic (optional, for complex apps)
├── config.py         # Settings via Pydantic Settings
├── db.py             # Database initialization
├── main.py           # App factory
└── utils/            # Shared utilities
test/
├── conftest.py       # Fixtures & shared test config
├── test_*.py         # Test files matching source modules
└── __init__.py
```

### Layer Responsibilities

| Layer | Responsibility |
|-------|---------------|
| **API** | Route definitions, request/response handling, validation |
| **Models** | ORM definitions, Pydantic schemas |
| **Services** | Business logic, orchestration |
| **Config** | Environment-based settings |
| **DB** | Connection setup, migrations |

## FastAPI Best Practices

### App Factory Pattern

```python
# app/main.py
from fastapi import FastAPI

def create_application() -> FastAPI:
    app = FastAPI(title="My API", version="0.1.0")
    app.include_router(api.router, prefix="/api")
    return app

app = create_application()
```

### Dependency Injection

```python
# app/api/dependencies.py
from fastapi import Depends
from tortoise import Tortoise

async def get_db():
    yield Tortoise.db
```

### Pydantic Schemas

```python
# app/models/pydantic.py
from pydantic import BaseModel, AnyHttpUrl, Field

class SummaryCreate(BaseModel):
    url: AnyHttpUrl

class SummaryResponse(SummaryCreate):
    id: int
    summary: str
    created_at: datetime

class SummaryUpdate(BaseModel):
    summary: str = Field(min_length=1)
```

### Error Handling

```python
# app/api/exceptions.py
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse

async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )
```

## TDD Workflow

### Test Structure

```python
# test/test_summaries.py

def test_create_summary(test_app_with_db):
    # Arrange
    payload = {"url": "https://example.com"}

    # Act
    response = test_app_with_db.post("/summaries/", json=payload)

    # Assert
    assert response.status_code == 201
    data = response.json()
    assert data["url"] == "https://example.com/"
    assert "id" in data
```

### Fixtures (conftest.py)

```python
# test/conftest.py
import pytest
from starlette.testclient import TestClient
from tortoise.contrib.fastapi import register_tortoise

from app.config import Settings, get_settings
from app.main import create_application

@pytest.fixture
def test_app():
    app = create_application()
    app.dependency_overrides[get_settings] = lambda: Settings(testing=True)
    with TestClient(app) as client:
        yield client

@pytest.fixture
def test_app_with_db():
    app = create_application()
    app.dependency_overrides[get_settings] = lambda: Settings(testing=True)
    register_tortoise(
        app,
        db_url=os.environ["DATABASE_TEST_URL"],
        modules={"models": ["app.models.tortoise"]},
        generate_schemas=True,
    )
    with TestClient(app) as client:
        yield client
    register_tortoise(app, generate_schemas=False)  # cleanup
```

## Async & Background Tasks

```python
# app/api/summaries.py
from fastapi import APIRouter, BackgroundTasks

router = APIRouter()

@router.post("/", response_model=SummaryResponse, status_code=201)
async def create_summary(
    payload: SummaryCreate,
    background_tasks: BackgroundTasks,
):
    summary_id = await crud.create(payload)
    background_tasks.add_task(generate_summary, summary_id, str(payload.url))
    return {"id": summary_id, "url": payload.url}
```

For heavy/background work, consider Celery or Arq instead of FastAPI background tasks.

## Database (Tortoise ORM + Aerich)

### Model Definition

```python
# app/models/tortoise.py
from tortoise import fields, models

class TextSummary(models.Model):
    id = fields.IntField(pk=True)
    url = fields.TextField()
    summary = fields.TextField(null=True)
    created_at = fields.DatetimeField(auto_now_add=True)

    def __str__(self):
        return self.url
```

### Pydantic Model from Tortoise

```python
from tortoise.contrib.pydantic import pydantic_model_creator

SummarySchema = pydantic_model_creator(TextSummary, name="Summary")
```

### Migrations

```bash
# Create migration
uv run aerich migrate --name "add_summary_field"

# Apply
uv run aerich upgrade

# Rollback
uv run aerich downgrade -n 1
```

## Testing Guidelines

### Test Categories

| Category | Purpose | Example |
|----------|---------|---------|
| **Unit** | Test pure functions & business logic | `test_services.py` |
| **Integration** | Test API endpoints with DB | `test_summaries.py` |
| **E2E** | Test full workflow | `test_e2e.py` |

### Assertions

```python
# Status codes
assert response.status_code == 201

# Response structure
data = response.json()
assert "id" in data
assert data["url"] == expected_url

# Error responses
assert response.status_code == 404
assert response.json()["detail"] == "Resource not found"
```

### Test Database

- Use a separate test database (`DATABASE_TEST_URL`)
- Reset state between tests using fixtures
- Mock external services (HTTP calls, email, etc.)

## Code Quality

### Ruff Configuration

```toml
# pyproject.toml
[tool.ruff]
line-length = 119
target-version = "py314"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "UP"]
ignore = ["B008"]  # FastAPI Depends() pattern

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### Commands

```bash
# Lint
uv run ruff check .

# Auto-fix
uv run ruff check . --fix

# Format
uv run ruff format .

# Check formatting
uv run ruff format --check .

# Test
uv run pytest

# Coverage
uv run pytest --cov=app --cov-report=html
```

## Docker & Deployment

### Development

```bash
docker compose up --build
```

### Production

```dockerfile
# Dockerfile.prod
FROM python:3.14-slim-bookworm
WORKDIR /home/app
COPY pyproject.toml .
RUN pip install .
COPY . .
USER app
CMD gunicorn --bind 0.0.0.0:$PORT app.main:app -k uvicorn.workers.UvicornWorker
```

### Environment Variables

```bash
DATABASE_URL=postgres://user:pass@host:5432/db
DATABASE_TEST_URL=postgres://user:pass@host:5432/test_db
ENVIRONMENT=production
PORT=8000
```

## Python 3.14 Features to Leverage

- Pattern matching with `match`/`case`
- Improved type inference
- Better async performance
- Use `typing.TypeAlias` for type aliases
- Use `dataclasses` where appropriate

## Common Anti-Patterns to Avoid

| Anti-Pattern | Better Approach |
|-------------|-----------------|
| Blocking I/O in async handlers | Use `asyncio.to_thread()` or async libraries |
| Direct DB calls in routes | Use service/repository layer |
| Magic numbers/strings | Define constants or enums |
| Large monolithic routes | Split into smaller routers |
| Hardcoded config | Use Pydantic Settings |
| Eager loading relationships | Use `select_related()` / `prefetch_related()` |
| Ignoring type hints | Always annotate function signatures |

## Checklist for PRs

- [ ] Tests written and passing
- [ ] Code formatted with Ruff
- [ ] Type hints added
- [ ] No hardcoded secrets or config
- [ ] Error cases handled
- [ ] Documentation updated if needed
- [ ] Migrations reviewed (if DB changed)
- [ ] Performance considerations noted (if applicable)
