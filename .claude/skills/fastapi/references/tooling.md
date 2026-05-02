# Tooling & Libraries

> **Read when:** running the dev server, adding a dependency, calling out to HTTP, or bridging async ↔ blocking.

## 1. Standard Toolchain

| Concern | Tool | Notes |
|---|---|---|
| Package management | `uv` | Already used in the Dockerfile builder stage. Use it for local dev too. |
| Linting & formatting | `ruff` | Enable the FastAPI rule set. Run on pre-commit. |
| Type checking | `ty` | Project type checker of choice. |
| Dev server | `docker compose up api` | uvicorn --reload inside the container; mounts `./app` live. |
| Production server | `docker compose up` | Same image, full stack. No local Python runtime needed. |
| HTTP client | `httpx` | Sync + async. **Never** add `requests` to this project. |
| Async ↔ blocking bridge | `asyncer` | `asyncify(...)` to call blocking I/O from async; `syncify(...)` for the reverse. Prefer over `anyio.to_thread` / `asyncio.to_thread`. |

## 2. Docker Compose CLI

```bash
# Start API only (hot-reload active)
docker compose up api

# Start full stack
docker compose up

# Rebuild after dependency changes
docker compose up --build api

# Run a one-off command inside the running api container
docker compose exec api alembic upgrade head
docker compose exec api python -m app.user.seed

# View logs
docker compose logs -f api
docker compose logs -f celery_worker
```

The `api` container is configured with `--reload --reload-dir /app/app`;
`./app` is bind-mounted read-only so edits are reflected instantly.

## 4. Asyncer Examples

```python
from asyncer import asyncify, syncify

# Blocking call inside async route
@router.get("/report")
async def generate_report() -> ReportResponse:
    pdf_bytes = await asyncify(render_pdf_blocking)(template="invoice")
    return ReportResponse(size=len(pdf_bytes))

# Async call inside Celery task (sync context)
@celery_app.task
def send_email_task(user_id: str) -> None:
    syncify(send_email_async)(user_id=user_id)
```

## 5. HTTPX Sketch

```python
import httpx

# Async usage — preferred inside FastAPI routes
async with httpx.AsyncClient(timeout=10) as client:
    resp = await client.get("https://api.example.com/x")
    resp.raise_for_status()
    data = resp.json()

# Sync usage — fine in Celery tasks
with httpx.Client(timeout=10) as client:
    resp = client.get("https://api.example.com/x")
```

Reuse a long-lived `AsyncClient` via a `dependencies.py` factory when calling
the same upstream from many routes.
