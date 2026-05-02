# Tooling & Libraries

> **Read when:** running the dev server, adding a dependency, calling out to HTTP, or bridging async ↔ blocking.

## 1. Standard Toolchain

| Concern | Tool | Notes |
|---|---|---|
| Package management | `uv` | Already used in the Dockerfile builder stage. Use it for local dev too. |
| Linting & formatting | `ruff` | Enable the FastAPI rule set. Run on pre-commit. |
| Type checking | `ty` | Project type checker of choice. |
| Dev server | `fastapi dev` | Reads entrypoint from `[tool.fastapi] entrypoint` in `pyproject.toml`. |
| Production server | `fastapi run` | Same entrypoint. Docker remains the deploy target. |
| HTTP client | `httpx` | Sync + async. **Never** add `requests` to this project. |
| Async ↔ blocking bridge | `asyncer` | `asyncify(...)` to call blocking I/O from async; `syncify(...)` for the reverse. Prefer over `anyio.to_thread` / `asyncio.to_thread`. |

## 2. `pyproject.toml` Entrypoint

```toml
[tool.fastapi]
entrypoint = "app.core.main:app"
```

With this in place, `fastapi dev` and `fastapi run` work without an explicit path.

## 3. CLI

```bash
# Dev (auto-reload)
fastapi dev 

# Production
fastapi run

```

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
