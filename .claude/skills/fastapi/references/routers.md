# Router Registration

> **Read when:** registering a new module router or adding shared dependencies to a router.

## 1. Where prefix and tags go

Declare `prefix` and `tags` on the `APIRouter()` inside the module — **never**
pass them to `include_router()`. Legacy entries in `app/apis/v1.py` that still
use `include_router(prefix=..., tags=...)` are an anti-pattern; refactor them
when you touch those files.

```python
# ✅ Correct — module file
user_router = APIRouter(prefix="/users", tags=["users"])

# ✅ Correct — apis/v1.py
router.include_router(user_router)
```

## 2. `app/apis/v1.py` — The Router Registry

```python
from fastapi import APIRouter

from app.user.routes import auth_router, user_router, user_management_router
from app.activity.routes import router as activity_router
from app.release_notes.routes import router as release_notes_router
# from app.<feature>.routes import <feature>_router  ← add new modules here

router = APIRouter()

router.include_router(auth_router)
router.include_router(user_router)
router.include_router(user_management_router)
router.include_router(activity_router)
router.include_router(release_notes_router)
# router.include_router(<feature>_router)
```

**Mounting prefix** is applied in `core/main.py`:

```python
app.include_router(api_v1_router, prefix=settings.API_V1_PREFIX)  # /api/v1
```

## 3. Router-Level Shared Dependencies

When every route in a router needs the same guard (a permission check, a feature
flag, an audit hook), declare it on the `APIRouter()` itself instead of repeating
`Depends(...)` in every signature.

```python
# ✅ Preferred — guard once at the router level
admin_router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_permission("admin:access"))],
)

@admin_router.get("/users")
async def list_users(session: SessionDep) -> list[UserResponse]:
    ...

# ⚠️ Avoid — same guard repeated on every route
@admin_router.get("/users")
async def list_users(
    _: Annotated[None, Depends(require_permission("admin:access"))],
    session: SessionDep,
): ...
```

Per-route permissions (`leads:view` vs `leads:edit` etc.) still go in the route
signature — only hoist guards that genuinely apply to **every** route in the router.
