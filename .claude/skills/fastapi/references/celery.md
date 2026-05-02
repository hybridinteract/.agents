# Celery Background Tasks

> **Read when:** adding or running a background task.

## 1. Task Location

- **Infrastructure** (Celery app config, base task class): `app/core/background/`
- **Module-specific tasks**: `app/<module>/tasks.py`

## 2. Celery App Path

Always reference the celery app as:

```bash
celery -A app.core.background.celery_app:celery_app worker
```

**Not** `app.core.celery_app` — that's the old path and causes import errors.

## 3. Async Tasks

Use the async-compatible session from `app/core/background/internals/session.py` —
**not** the FastAPI `get_session` dependency — for Celery tasks.

```python
# app/<feature>/tasks.py
from app.core.background.celery_app import celery_app
from app.core.background.internals.session import get_async_session

@celery_app.task
def process_lead_task(lead_id: str) -> None:
    # If the task body is async, bridge with asyncer.syncify
    from asyncer import syncify
    syncify(_process_lead_async)(lead_id)

async def _process_lead_async(lead_id: str) -> None:
    async with get_async_session() as session:
        ...
```

See `tooling.md` for the `asyncer` async↔sync bridge.
