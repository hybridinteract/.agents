# Architecture Patterns

> **Read when:** writing or refactoring routes, services, CRUDs, or models.

## 1. Layered Architecture (strict — never skip layers)

```
Route (FastAPI router)
  └── Service (business logic, transaction boundary)
        └── CRUD (DB operations, no commit)
              └── Model (SQLAlchemy ORM)
```

**Rules:**
- **Routes** inject dependencies, parse request, call service, return response.
- **Services** own `session.commit()` — never in CRUD or routes.
- **CRUD** uses `session.flush()` + `session.refresh()` — never `commit()`.
- **Models** are pure data containers — no business logic.
- **Always `async def`** — every route, service, and CRUD function. The DB layer
  (`get_session`, `CRUDBase`, helpers) is async end-to-end. The "default to `def`
  when in doubt" rule from the upstream FastAPI skill does **not** apply here.

## 2. Transaction Ownership

```python
# ✅ Correct — service commits
async def create_user(session: AsyncSession, data: UserCreate) -> User:
    user = await self._user_crud.create(session, obj_in=data)
    await session.commit()          # ← service owns this
    return user

# ❌ Wrong — CRUD commits
async def create(self, session, obj_in):
    ...
    await session.commit()          # ← NEVER in CRUD
```

## 3. Generic CRUDBase

Define the CRUD class in `crud/<model>_crud.py`. **Do NOT instantiate a module-level
singleton.** All CRUDs and services are wired through `dependencies.py` using
`@lru_cache` factories (one instance per process), then injected via `Annotated`
aliases. Full DI recipe is in `dependency-injection.md`.

```python
# app/<feature>/crud/user_crud.py
class UserCRUD(CRUDBase[User, UserCreate, UserUpdate]):
    # Override only what's different
    async def get_by_email(self, session: AsyncSession, email: str) -> User | None:
        ...
```

```python
# app/<feature>/dependencies.py
from functools import lru_cache
from typing import Annotated
from fastapi import Depends

from app.<feature>.crud.user_crud import UserCRUD

@lru_cache
def get_user_crud() -> UserCRUD:
    return UserCRUD(User)

UserCRUDDep = Annotated[UserCRUD, Depends(get_user_crud)]
```

```python
# app/<feature>/services/user_service.py
class UserService:
    def __init__(self, user_crud: UserCRUD) -> None:
        self._user_crud = user_crud   # ← injected, never imported as a singleton

    async def create_user(self, session: AsyncSession, data: UserCreate) -> User:
        user = await self._user_crud.create(session, obj_in=data)
        await session.commit()
        return user
```

Services receive their CRUD collaborators via the constructor. They never import
or instantiate CRUD singletons themselves.

## 4. Module Public API (`__init__.py`)

Each module exposes only what consumers need:

```python
# app/user/__init__.py
from .models import User
from .routes import auth_router, user_router, user_management_router

__all__ = ["User", "auth_router", "user_router", "user_management_router"]
```
