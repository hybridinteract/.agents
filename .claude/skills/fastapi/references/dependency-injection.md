# Dependency Injection Patterns

> **Read when:** wiring a CRUD or service, defining shared `Depends()`, or working with `yield` cleanup logic.

## 1. Core Aliases

Define `Annotated` type aliases once per module (or in `dependencies.py`) and
reuse them across routes:

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_session
from app.user.models import User
from app.user.auth_management import get_current_active_user
from app.user.permission_management import require_permission

SessionDep     = Annotated[AsyncSession, Depends(get_session)]
CurrentUserDep = Annotated[User, Depends(get_current_active_user)]

# Standard usage
async def route(session: SessionDep): ...
async def route(current_user: CurrentUserDep): ...

# Permission guard (require_permission returns None — name it _)
async def route(
    _: Annotated[None, Depends(require_permission("leads:view"))],
    current_user: CurrentUserDep,
    session: SessionDep,
): ...
```

## 2. CRUD & Service Factory Pattern (Project Standard)

Every CRUD class and service is wired through `app/<feature>/dependencies.py`
using `@lru_cache` factories. This produces one instance per process, makes
construction explicit, and keeps services free of singleton imports.

**Rules:**
- Module-level singletons (`user_crud = UserCRUD(User)`) are forbidden.
- Services receive their CRUD collaborators via the constructor.
- Service files never import another service or CRUD as a singleton — they
  declare it as a constructor parameter and rely on the factory wiring it.
- All wiring lives in `dependencies.py`. Routes import only the `*Dep` aliases.

```python
# app/<feature>/dependencies.py
from functools import lru_cache
from typing import Annotated
from fastapi import Depends

from app.<feature>.crud.user_crud import UserCRUD
from app.<feature>.services.user_service import UserService
from app.user.models import User

@lru_cache
def get_user_crud() -> UserCRUD:
    return UserCRUD(User)

UserCRUDDep = Annotated[UserCRUD, Depends(get_user_crud)]

@lru_cache
def _build_user_service(user_crud: UserCRUD) -> UserService:
    return UserService(user_crud=user_crud)

def get_user_service(user_crud: UserCRUDDep) -> UserService:
    return _build_user_service(user_crud)

UserServiceDep = Annotated[UserService, Depends(get_user_service)]
```

```python
# app/<feature>/routes/user_routes.py
@user_router.post("/")
async def create_user(
    data: UserCreate,
    session: SessionDep,
    user_service: UserServiceDep,
) -> UserResponse:
    return await user_service.create_user(session, data)
```

## 3. Dependency `yield` Scopes

Dependencies declared with `yield` accept a `scope` argument:

- `scope="request"` (default) — exit code runs **after** the response is sent.
  Right for DB sessions, file handles, anything where cleanup is non-blocking
  on the client.
- `scope="function"` — exit code runs **after** response data is generated but
  **before** it is sent. Use when cleanup must complete before the client
  observes the response (e.g., committing an audit row that must be visible
  if the client immediately re-queries).

```python
def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        db.close()

DBDep = Annotated[DBSession, Depends(get_db)]   # request scope (default)

def get_audit_ctx():
    try:
        yield AuditContext()
    finally:
        flush_audit_buffer()

AuditDep = Annotated[AuditContext, Depends(get_audit_ctx, scope="function")]
```

## 4. Avoid Class Dependencies

Don't pass a class directly to `Depends()`. Wrap it in a function that returns
an instance — function dependencies compose better with `@lru_cache` and the
factory pattern above.

```python
# ✅ Correct
@dataclass
class Paginator:
    offset: int = 0
    limit: int = 100

def get_paginator(offset: int = 0, limit: int = 100) -> Paginator:
    return Paginator(offset=offset, limit=limit)

PaginatorDep = Annotated[Paginator, Depends(get_paginator)]

# ❌ Wrong — class as a dependency
async def route(p: Annotated[Paginator, Depends()]): ...
```
