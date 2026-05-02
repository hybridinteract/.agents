# Operations — Exceptions, Logging & Docker

> **Read when:** adding an exception class, instrumenting logs, or touching Docker / docker-compose.

## 1. Module Exceptions

Each module has its own `exceptions.py` with domain-specific exceptions as
`HTTPException` subclasses:

```python
# app/lead/exceptions.py
from fastapi import HTTPException, status

class LeadNotFoundException(HTTPException):
    def __init__(self, lead_id: str):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Lead {lead_id} not found",
        )
```

Raise these from services; routes do not need to catch them — global handlers
in `app/core/exceptions.py` translate them to JSON responses.

## 2. Global Handlers

`app/core/exceptions.py` registers global handlers on the FastAPI app:

| Exception | Status |
|---|---|
| `RequestValidationError` | 422 |
| `ValidationError` (Pydantic) | 422 |
| `IntegrityError` (SQLAlchemy) | 409 |
| `OperationalError` | 503 |
| `SQLAlchemyError` | 500 |
| `ValueError` | 400 |
| `PermissionError` | 403 |
| `Exception` (catch-all) | 500 |

## 3. Logging

```python
from app.core.logging import get_logger

logger = get_logger(__name__)

logger.info("User created", extra={"user_id": str(user.id)})
logger.warning("Slow query detected")
logger.error("DB write failed", exc_info=True)
```

**Log files:**
- `logs/<APP_NAME>.log` — all logs, rotating by size.
- `logs/<APP_NAME>_errors.log` — errors only, rotating by size.
- Console output is colored in DEBUG mode.

## 4. Docker Setup

### Services

| Service | Image | Port |
|---------|-------|------|
| `api` | custom Dockerfile | `${API_PORT:-8000}` |
| `postgres` | `postgres:16-alpine` | `${POSTGRES_PORT:-5432}` |
| `redis` | `redis:7-alpine` | `${REDIS_PORT:-6379}` |
| `celery_worker` | custom Dockerfile | — |
| `flower` | custom Dockerfile | `${FLOWER_PORT:-5555}` |

### Network & Volume Naming

Always prefix with project name:

```yaml
networks:
  <project>_network:
    name: <project>_network

volumes:
  <project>_postgres_data:
    name: <project>_postgres_data
```

### Dockerfile Pattern (multi-stage)

- **Stage 1 (builder)**: `python:3.13-slim` + `uv pip install`.
- **Stage 2 (runtime)**: `python:3.13-slim` + non-root `appuser` (uid 1000).
