# Herbally Backend — Project Conventions

> Authoritative guide for building and scaling modules in the Herbally hospital management system.

---

## 0. Skill Defaults Overridden by This Project

The FastAPI skill (`SKILL.md`) is the framework baseline. This project diverges in
the following ways — **these conventions win** wherever they conflict.

| Topic | Skill default | This project |
|---|---|---|
| SQL ORM | SQLModel | Async SQLAlchemy 2.x + `CRUDBase[Model, Create, Update]` (§3.3, §5) |
| Route style | `def` when in doubt | `async def` — entire DB layer is async (§3.1) |
| CRUD/service wiring | Module-level singletons | `@lru_cache` factories in `dependencies.py` (§3.3, §15) |
| Router prefix/tags | On `APIRouter()` | Same — but legacy `apis/v1.py` still passes them in `include_router()`; refactor opportunistically (§7) |
| HTTP client | HTTPX (skill recommends) | HTTPX — adopted (§16) |
| Async/blocking bridge | Asyncer (skill recommends) | Asyncer — adopted (§16) |

Everything else in the FastAPI skill applies as written.

---

## 1. Project Structure

```
<project_name>/
├── app/
│   ├── __init__.py
│   ├── core/                        # ✅ Reusable — never project-specific
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app factory + lifespan
│   │   ├── settings.py              # Pydantic BaseSettings configuration
│   │   ├── database.py              # Async SQLAlchemy engine + session
│   │   ├── models.py                # Base declarative class
│   │   ├── schemas.py               # ListParams + SortOrder (list-endpoint base)
│   │   ├── exceptions.py            # Global exception handlers
│   │   ├── middleware.py            # CORS, GZip, TrustedHost, timing
│   │   ├── logging.py               # Rotating file + colored console logging
│   │   ├── metrics.py               # Prometheus instrumentator
│   │   ├── utils.py                 # utc_now() and other shared helpers
│   │   ├── alembic_models_import.py # Single file to import all models for Alembic
│   │   ├── crud/                    # Generic CRUD + pagination helpers
│   │   │   ├── __init__.py          # Re-exports CRUDBase, apply_sorting, paginated_select
│   │   │   ├── base.py              # CRUDBase[Model, Create, Update]
│   │   │   ├── helpers.py           # apply_sorting(), paginated_select()
│   │   │   ├── README.md            # 📖 Adoption guide for list endpoints
│   │   │   └── SEARCH_STRATEGY.md   # 📖 Search technique decision guide + indexing strategy
│   │   ├── background/              # Celery app + task infrastructure
│   │   │   ├── celery_app.py
│   │   │   ├── tasks.py
│   │   │   └── internals/           # Base task, context, retry, monitoring
│   │   ├── cache/                   # Redis cache abstraction
│   │   │   └── cache.py
│   │   └── object_storage/          # S3-compatible file storage
│   │       ├── storage.py
│   │       └── utils.py
│   │
│   ├── apis/
│   │   └── v1.py                    # Aggregates all module routers
│   │
│   ├── user/                        # ✅ Reusable — auth + RBAC
│   │   ├── __init__.py
│   │   ├── models.py                # User, Role, Permission, RefreshToken
│   │   ├── exceptions.py
│   │   ├── seed.py                  # Idempotent role/permission seeder
│   │   ├── create_admin.py          # Interactive super-admin CLI
│   │   ├── auth_management/         # JWT login, refresh, logout
│   │   ├── permission_management/   # RBAC scoped access helpers
│   │   ├── crud/                    # user_crud, role_crud, permission_crud, refresh_token_crud
│   │   ├── schemas/                 # user_schemas, admin_schemas
│   │   ├── services/                # user_service, admin_service, user_query_service
│   │   └── routes/                  # user_routes, admin_routes
│   │
│   ├── activity/                    # ✅ Reusable — append-only audit log
│   │   ├── models.py
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── crud.py
│   │
│   ├── release_notes/               # ✅ Reusable — What's New system
│   │   ├── models.py
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── crud.py
│   │
│   └── <feature>/                   # 🔧 Project-specific feature modules
│       ├── __init__.py              # Module public API
│       ├── models/                  # DB models (sub-package if complex)
│       ├── schemas/                 # Pydantic schemas (sub-package if complex)
│       ├── crud/                    # CRUD classes (sub-package if complex)
│       ├── services/                # Business logic (sub-package if complex)
│       ├── routes/                  # FastAPI routers (sub-package if complex)
│       ├── dependencies.py          # FastAPI Depends() helpers
│       ├── exceptions.py            # Module-specific exceptions
│       ├── enums.py                 # Module-specific enums
│       ├── permissions.py           # Permission constants for the module
│       └── tasks.py                 # Celery tasks for the module
│
├── migrations/
│   ├── env.py                       # Alembic async env config
│   ├── script.py.mako
│   └── versions/                   # Timestamped migration files
│
├── docker/
│   ├── Dockerfile                   # Multi-stage Python build
│   ├── docker-entrypoint.sh
│   ├── celery-worker-entrypoint.sh
│   └── flower-entrypoint.sh
│
├── docs/                            # Module-level documentation
├── logs/                            # Runtime log files (gitignored)
├── alembic.ini
├── docker-compose.yml
├── pyproject.toml
├── .env.example
└── README.md
```

---

## 1.1 Access Control (RBAC + Scope)

Access control is split into two independent layers — see
[RBAC_AND_SCOPE.md](./RBAC_AND_SCOPE.md) for the full spec.

- **Permission layer** (`require_permission("resource:action")`) — gates
  whether a user can call an endpoint.
- **Scope layer** (`Depends(get_request_scope([...]))` → `ScopeContext`) —
  filters the rows the endpoint returns/touches.

Every scoped resource has a paired `<resource>:read_all` override
permission. The 13 baseline roles (super_admin → employee) are seeded by
[`app/user/seed.py`](../app/user/seed.py).

When adding a new scoped resource, follow the recipe in
[RBAC_AND_SCOPE.md §7](./RBAC_AND_SCOPE.md#7-adding-a-new-scoped-resource--recipe).

---

## 2. Naming Conventions

### Files & Directories
| Type | Convention | Example |
|------|-----------|---------|
| Python files | `snake_case.py` | `user_crud.py`, `admin_service.py` |
| Module directories | `snake_case/` | `app/lead/`, `app/release_notes/` |
| Migration files | `YYYY_MM_DD_HHMM-<rev>_<slug>.py` | `2026_02_09_0657-8600ba4ec5f7_user_init_models.py` |
| Docker scripts | `kebab-case.sh` | `celery-worker-entrypoint.sh` |

### Python Identifiers
| Type | Convention | Example |
|------|-----------|---------|
| Classes | `PascalCase` | `UserService`, `LeadCRUD` |
| Functions/methods | `snake_case` | `get_current_user`, `create_lead` |
| Constants | `UPPER_SNAKE_CASE` | `ROLES`, `PERMISSIONS`, `API_V1_PREFIX` |
| Pydantic models | `PascalCase` with suffix | `UserCreate`, `UserUpdate`, `UserResponse` |
| CRUD factories | `get_<model>_crud` | `get_user_crud`, `get_role_crud` |
| Service factories | `get_<name>_service` | `get_user_service`, `get_lead_service` |
| CRUD/service DI aliases | `<Model>CRUDDep`, `<Name>ServiceDep` | `UserCRUDDep`, `LeadServiceDep` |
| Router instances | `<module>_router` | `auth_router`, `user_router`, `lead_router` |

### Database
| Type | Convention | Example |
|------|-----------|---------|
| Table names | `plural_snake_case` | `users`, `refresh_tokens`, `role_permissions` |
| Index names | `ix_<table>_<columns>` | `ix_users_status`, `ix_activity_logs_actor_id` |
| FK constraint | SQLAlchemy default | SQLAlchemy handles naming |
| Association tables | `<table1>_<table2>` | `user_roles`, `role_permissions` |
| Enum type names | Keep SQLAlchemy default | Uses class name |

### Permissions
```
<resource>:<action>          →  "leads:view"
<resource>:<action>:<scope>  →  "leads:view:all", "leads:view:team"
```

---

## 3. Architecture Patterns

### 3.1 Layered Architecture (strict — never skip layers)

```
Route (FastAPI router)
  └── Service (business logic, transaction boundary)
        └── CRUD (DB operations, no commit)
              └── Model (SQLAlchemy ORM)
```

**Rules:**
- **Routes** inject dependencies, parse request, call service, return response
- **Services** own `session.commit()` — never in CRUD or routes
- **CRUD** uses `session.flush()` + `session.refresh()` — never `commit()`
- **Models** are pure data containers — no business logic inside models
- **Always `async def`** — every route, service, and CRUD function. The DB layer
  (`get_session`, `CRUDBase`, helpers) is async end-to-end. The "default to `def`
  when in doubt" rule from the FastAPI skill does **not** apply to this project.

### 3.2 Transaction Ownership

```python
# ✅ Correct — service commits
async def create_user(session: AsyncSession, data: UserCreate) -> User:
    user = await user_crud.create(session, obj_in=data)
    await session.commit()          # ← service owns this
    return user

# ❌ Wrong — CRUD commits
async def create(self, session, obj_in):
    ...
    await session.commit()          # ← NEVER in CRUD
```

### 3.3 Generic CRUDBase

Define the CRUD class in `crud/<model>_crud.py`. **Do NOT instantiate a module-level
singleton.** All CRUDs and services are wired through `dependencies.py` using
`@lru_cache` factories (one instance per process), then injected via `Annotated` aliases.

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

Services receive their CRUD collaborators via the constructor. They never
import or instantiate CRUD singletons themselves. See §15 for the full DI recipe.

### 3.4 Module Public API (`__init__.py`)

Each module exposes only what consumers need:

```python
# app/user/__init__.py
from .models import User
from .routes import auth_router, user_router, user_management_router

__all__ = ["User", "auth_router", "user_router", "user_management_router"]
```

---

### 3.5 List Endpoints — Filtering, Sorting & Pagination

Every list endpoint that exposes filters, sort, and a total row count **must** follow
this standard. See `app/core/crud/README.md` for the full step-by-step recipe.

#### The five-layer flow

```
Route (XxxListParams = Depends())
  └─ Service (thin pass-through + any business-rule overrides)
       └─ CRUD.get_list_filtered(session, skip, limit, sort_by, sort_order, …filters)
            └─ paginated_select(session, base_query, skip, limit, order_clauses)
                 └─ SQL: SELECT …, COUNT(*) OVER () OFFSET … LIMIT …
                 └─ Returns (items: list, total: int)
```

#### Required pieces per module

| File | What to add |
|---|---|
| `enums.py` | `XxxSortField(str, Enum)` — the allowed sort columns |
| `schemas/xxx_schemas.py` | `XxxListParams(ListParams)` — override `sort_by` + add filter fields |
| `crud/xxx_crud.py` | `get_list_filtered(…)` — builds filters, calls `paginated_select()` |
| `services/xxx_service.py` | `list_xxx(session, params, owner_user_ids)` — pass-through + rules |
| `routes/xxx_routes.py` | `params: XxxListParams = Depends()` — scope injection |

#### `XxxListParams` schema rules

```python
from app.core.schemas import ListParams
from app.mymodule.enums import ThingSortField, SortDirection

class ThingListParams(ListParams):
    # Override with typed enum — FastAPI returns 422 on invalid values
    # and Swagger renders a dropdown automatically.
    sort_by:    ThingSortField = ThingSortField.CREATED_AT
    sort_order: SortDirection  = SortDirection.DESC
    limit: int = Field(50, ge=1, le=500)   # override default if needed

    # Filter fields
    search:    Optional[str]  = None
    status:    Optional[str]  = None
    date_from: Optional[date] = None
    date_to:   Optional[date] = None
```

#### Scope enforcement in routes

Pydantic models are immutable — never mutate `params` fields. Use a local variable:

```python
from typing import Annotated

@router.get("/")
async def list_things(
    params: Annotated[ThingListParams, Depends()],
    session: SessionDep,
    current_user: CurrentUserDep,
) -> ThingListResponse:
    owner_user_ids = None
    if not current_user.has_permission("things:read_all"):
        owner_user_ids = [current_user.id]   # ← local var, not params mutation

    return await thing_service.list_things(session, params, owner_user_ids)
```

#### `paginated_select()` — single-query pagination

Prefer `paginated_select()` over the two-query (`COUNT(*) + SELECT`) pattern.
It emits one SQL round-trip using `COUNT(*) OVER ()`, evaluating `WHERE`/`JOIN`
clauses exactly once:

```python
# ✅ Preferred — one query
items, total = await paginated_select(
    session, base_query,
    skip=skip, limit=limit,
    order_clauses=[Thing.created_at.desc(), Thing.id.desc()],
)

# ⚠️ Acceptable for simple cases, but two queries
total = (await session.execute(count_query)).scalar() or 0
items = list((await session.execute(items_query)).scalars().all())
```

#### Sort column tiebreaker

Always append `model.id.desc()` as the last element of `order_clauses`.
Without it, `OFFSET`/`LIMIT` can return duplicate or skipped rows when
rows are inserted between page requests on a non-unique sort key.

---

### 3.6 FastAPI Parameter Declarations

These rules come straight from the FastAPI skill — apply them in every route.

**Always use `Annotated` for `Path`, `Query`, `Header`, `Body`, `Form`, `File`, `Cookie`.**

```python
# ✅ Correct
@router.get("/items/{item_id}")
async def read_item(
    item_id: Annotated[int, Path(ge=1, description="Item ID")],
    q: Annotated[str | None, Query(max_length=50)] = None,
): ...

# ❌ Wrong — old default-value style
async def read_item(
    item_id: int = Path(ge=1, description="Item ID"),
    q: str | None = Query(default=None, max_length=50),
): ...
```

**Never use Ellipsis (`...`) for required fields.** A field without a default *is*
required.

```python
# ✅ Correct
class Item(BaseModel):
    name: str
    price: float = Field(gt=0)

# ❌ Wrong
class Item(BaseModel):
    name: str = ...
    price: float = Field(..., gt=0)
```

**No Pydantic `RootModel`.** Use `Annotated` + validators directly:

```python
# ✅ Correct
@router.post("/items/")
async def create_items(items: Annotated[list[int], Field(min_length=1), Body()]):
    return items
```

**One HTTP method per function.** Never use `@router.api_route(methods=[...])`.
Split into `@router.get(...)` + `@router.post(...)` etc.

---

### 3.7 Response Handling

**Prefer return-type annotation over `response_model=`.** The return type is what
filters and serializes the response (Pydantic-Rust path).

```python
# ✅ Preferred — return type drives validation, filtering, serialization
@router.get("/me")
async def get_me(current_user: CurrentUserDep) -> UserResponse:
    return UserResponse.model_validate(current_user)
```

Use `response_model=` **only** when the actual returned type and the documented
type genuinely differ (e.g., returning an internal model that must be filtered
to a public schema):

```python
@router.get("/me", response_model=UserPublic)
async def get_me(current_user: CurrentUserDep) -> Any:
    return current_user   # InternalUser → filtered down to UserPublic
```

**Do NOT use `ORJSONResponse` or `UJSONResponse`** — both are deprecated. Pydantic
v2 already serializes via Rust at native speed.

---

### 3.8 Streaming

For any streaming endpoint, follow these patterns from the skill.

**JSON Lines** — declare the return type, `yield` items:

```python
from collections.abc import AsyncIterable

@router.get("/items/stream")
async def stream_items(session: SessionDep) -> AsyncIterable[ItemResponse]:
    async for item in item_crud.iter_all(session):
        yield ItemResponse.model_validate(item)
```

**Server-Sent Events** — `fastapi.sse.EventSourceResponse`. Yield plain Pydantic
models (auto-serialized to `data:`), or `ServerSentEvent` for full control over
`event` / `id` / `retry`:

```python
from fastapi.sse import EventSourceResponse, ServerSentEvent

@router.get("/events", response_class=EventSourceResponse)
async def stream_events() -> AsyncIterable[ServerSentEvent]:
    yield ServerSentEvent(data={"status": "started"}, event="status", id="1")
    yield ServerSentEvent(data={"progress": 50}, event="progress", id="2")
```

**Byte streams** — subclass `StreamingResponse` (set `media_type`), pass via
`response_class=`, and `yield`/`yield from` from the function. Do not return
`StreamingResponse(...)` directly:

```python
from fastapi.responses import StreamingResponse

class PNGStreamingResponse(StreamingResponse):
    media_type = "image/png"

@router.get("/image", response_class=PNGStreamingResponse)
def stream_image():   # plain def is fine for blocking file I/O
    with read_image() as f:
        yield from f
```

---

## 4. Settings & Configuration

### 4.1 Pydantic Settings Class (`app/core/settings.py`)

- Use `BaseSettings` with `.env` file support
- Group fields with section comments: `# ==================== Database Settings ====================`
- Computed values (built from other fields) use `@property`
- Required secrets have no default — will fail fast on startup:
  ```python
  SECRET_KEY: str = Field(..., description="Secret key for signing")
  ```
- Validate secrets at class level:
  ```python
  @field_validator("SECRET_KEY", "JWT_SECRET_KEY")
  @classmethod
  def validate_secret_keys(cls, v: str, info) -> str:
      if not v or len(v) < 32:
          raise ValueError(f"{info.field_name} must be at least 32 characters")
      return v
  ```
- Cache settings with `@lru_cache()` — only one instance per process

### 4.2 Environment Variables

All env vars must be documented in `.env.example`:
```bash
# ==================== Section Name ====================
VAR_NAME=default_value     # Inline comment explaining purpose
```

---

## 5. Database & Alembic

### 5.1 Model Conventions

```python
class MyModel(Base):
    __tablename__ = "plural_snake_case"
    __table_args__ = (
        Index('ix_mytable_field', 'field'),   # Always name indexes explicitly
    )

    # Primary key — always UUID
    id: Mapped[UUIDType] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    
    # Timestamps — always timezone=True, always UTC
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utc_now, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utc_now, onupdate=utc_now, nullable=False)
    
    # Soft delete pattern
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    deleted_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True), nullable=True)
```

### 5.2 Alembic Model Import

`app/core/alembic_models_import.py` — the **single source of truth** for Alembic's autogenerate:

```python
# Import Base for metadata
from app.core.models import Base

# User module
from app.user.models import User, Role, Permission, RefreshToken, UserRole, RolePermission

# Add every new module here ↓
from app.activity.models import ActivityLog
from app.release_notes.models import ReleaseNote
# from app.mymodule.models import MyModel
```

`migrations/env.py` does `from app.core.alembic_models_import import *` — add models only to the import file, never directly to env.py.

### 5.3 Migration File Naming

```
YYYY_MM_DD_HHMM-<rev>_<description>.py
2026_02_09_0657-8600ba4ec5f7_user_init_models.py
```

Configured in `alembic.ini`:
```ini
file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s
```

---

## 6. Authentication & RBAC

### 6.1 Permission Naming

```python
PERMISSIONS: list[tuple[str, str, str]] = [
    # (resource, action, description)
    ("leads", "view",     "View lead details"),
    ("leads", "view:all", "View all leads in the system"),
    ("users", "create",   "Create new user accounts"),
]
# Permission name stored in DB: "leads:view", "leads:view:all"
```

### 6.2 Role Definition

- Roles defined as constants in `user/seed.py`
- `is_system=True` → cannot be deleted via UI
- `super_admin` role bypasses ALL permission checks in the system

### 6.3 Seed Script (`user/seed.py`)

- **Idempotent** — safe to run multiple times; only inserts missing data
- Runs automatically on application startup (`lifespan` in `main.py`)
- Can also run manually: `python -m app.user.seed`
- Use `dispose_engine=False` when called at startup (engine shared with app)
- Keeps project-specific roles/permissions ONLY in seed.py

---

## 7. API Router Registration

Declare `prefix` and `tags` on the `APIRouter()` inside the module — never pass them to `include_router()`.

### `app/apis/v1.py` — The Router Registry

```python
from fastapi import APIRouter

from app.user.routes import auth_router, user_router, user_management_router
from app.activity.routes import router as activity_router
from app.release_notes.routes import router as release_notes_router
# from app.mymodule.routes import mymodule_router  ← add new modules here

router = APIRouter()

router.include_router(auth_router)
router.include_router(user_router)
router.include_router(user_management_router)
router.include_router(activity_router)
router.include_router(release_notes_router)
# router.include_router(mymodule_router)
```

**Mounting prefix** is applied in `core/main.py`:
```python
app.include_router(api_v1_router, prefix=settings.API_V1_PREFIX)  # /api/v1
```

### 7.1 Router-Level Shared Dependencies

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

---

## 8. Celery Background Tasks

### 8.1 Task Location

- Infrastructure (Celery app config, base task class): `app/core/background/`
- Module-specific tasks: `app/<module>/tasks.py`

### 8.2 Celery App Path

```python
# Always reference as:
celery -A app.core.background.celery_app:celery_app worker
# NOT: app.core.celery_app  (old path — causes import errors)
```

### 8.3 Async Tasks

Use the async-compatible session from `app/core/background/internals/session.py` — **not** the FastAPI `get_session` dependency — for Celery tasks.

---

## 9. Exception Handling

### 9.1 Module Exceptions

Each module has its own `exceptions.py` with domain-specific exceptions as `HTTPException` subclasses:

```python
# app/lead/exceptions.py
from fastapi import HTTPException, status

class LeadNotFoundException(HTTPException):
    def __init__(self, lead_id: str):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Lead {lead_id} not found"
        )
```

### 9.2 Global Handlers

`app/core/exceptions.py` registers global handlers on the FastAPI app for:
- `RequestValidationError` → 422
- `ValidationError` (Pydantic) → 422
- `IntegrityError` (SQLAlchemy) → 409
- `OperationalError` → 503
- `SQLAlchemyError` → 500
- `ValueError` → 400
- `PermissionError` → 403
- `Exception` (catch-all) → 500

---

## 10. Logging

### 10.1 Usage

```python
from app.core.logging import get_logger

logger = get_logger(__name__)

logger.info("User created", extra={"user_id": str(user.id)})
logger.warning("Slow query detected")
logger.error("DB write failed", exc_info=True)
```

### 10.2 Log Files

- `logs/<APP_NAME>.log` — all logs, rotating by size
- `logs/<APP_NAME>_errors.log` — errors only, rotating by size
- Console output colored in DEBUG mode

---

## 11. Docker Setup

### 11.1 Services

| Service | Image | Port |
|---------|-------|------|
| `api` | custom Dockerfile | `${API_PORT:-8000}` |
| `postgres` | `postgres:16-alpine` | `${POSTGRES_PORT:-5432}` |
| `redis` | `redis:7-alpine` | `${REDIS_PORT:-6379}` |
| `celery_worker` | custom Dockerfile | — |
| `flower` | custom Dockerfile | `${FLOWER_PORT:-5555}` |

### 11.2 Network & Volume Naming

Always prefix with project name:
```yaml
networks:
  <project>_network:
    name: <project>_network

volumes:
  <project>_postgres_data:
    name: <project>_postgres_data
```

### 11.3 Dockerfile Pattern

Multi-stage build:
- **Stage 1 (builder)**: `python:3.13-slim` + `uv pip install`
- **Stage 2 (runtime)**: `python:3.13-slim` + non-root `appuser` (uid 1000)

---

## 12. What Goes Where (Decision Guide)

| Question | Answer |
|----------|--------|
| Does this apply to every project? | `app/core/` |
| Is this authentication/user-management? | `app/user/` |
| Is this an audit trail of events? | `app/activity/` |
| Is this version change announcements? | `app/release_notes/` |
| Is this feature-specific business logic? | `app/<feature>/services/` |
| Does this define what DB rows look like? | `app/<feature>/models/` |
| Does this define the API contract shape? | `app/<feature>/schemas/` |
| Does this define who can do what? | `app/<feature>/permissions.py` + `user/seed.py` |
| Is this a background task? | `app/<feature>/tasks.py` |
| Does this define reusable FastAPI `Depends()`? | `app/<feature>/dependencies.py` |

---

## 13. Adding a New Feature Module

1. Create `app/<feature>/` directory with the standard sub-structure
2. Add models to `app/core/alembic_models_import.py`
3. Add permissions to `app/user/seed.py` (PERMISSIONS + ROLE_PERMISSIONS)
4. Register router in `app/apis/v1.py`
5. Run `alembic revision --autogenerate -m "<description>"`
6. Run `alembic upgrade head`
7. **For any list endpoint:** follow the recipe in `app/core/crud/README.md`
   — define `XxxSortField` enum → `XxxListParams(ListParams)` schema →
   `get_list_filtered()` CRUD method → service pass-through → route with
   `Depends()`. See §3.5 above for the short form.

---

## 14. Datetime Standards

- **All datetime fields**: `DateTime(timezone=True)` — stores with timezone
- **Default**: `default=utc_now` (from `app.core.utils`)
- **Never**: `datetime.utcnow()` — deprecated; use `datetime.now(timezone.utc)`
- **Frontend handles display timezone conversion** — backend always stores UTC

---

## 15. Dependency Injection Patterns

Define `Annotated` type aliases once per module (or in `dependencies.py`) and reuse them across routes:

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_session
from app.user.models import User
from app.user.auth_management import get_current_active_user
from app.user.permission_management import require_permission

# Module-level aliases — import these in route files
SessionDep     = Annotated[AsyncSession, Depends(get_session)]
CurrentUserDep = Annotated[User, Depends(get_current_active_user)]

# ✅ Standard DB session dependency
async def route(session: SessionDep): ...

# ✅ Current user dependency
async def route(current_user: CurrentUserDep): ...

# ✅ Permission guard (require_permission returns None — name it _)
async def route(
    _: Annotated[None, Depends(require_permission("leads:view"))],
    current_user: CurrentUserDep,
    session: SessionDep,
): ...
```

### 15.1 CRUD & Service Factory Pattern (Project Standard)

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

### 15.2 Dependency `yield` Scopes

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

### 15.3 Avoid Class Dependencies

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

---

## 16. Tooling & Libraries

| Concern | Tool | Notes |
|---|---|---|
| Package management | `uv` | Already used in the Dockerfile builder stage. Use it for local dev too. |
| Linting & formatting | `ruff` | Enable the FastAPI rule set. Run on pre-commit. |
| Type checking | `ty` | Project type checker of choice. |
| Dev server | `fastapi dev` | Reads entrypoint from `[tool.fastapi] entrypoint` in `pyproject.toml`. |
| Production server | `fastapi run` | Same entrypoint. The Docker setup remains the deploy target. |
| HTTP client | `httpx` | Sync + async. **Never** add `requests` to this project. |
| Async ↔ blocking bridge | `asyncer` | Use `asyncify(...)` to call blocking I/O from async; `syncify(...)` for the reverse. Prefer over `anyio.to_thread`/`asyncio.to_thread`. |

### 16.1 `pyproject.toml` Entrypoint

```toml
[tool.fastapi]
entrypoint = "app.core.main:app"
```

With this in place, `fastapi dev` and `fastapi run` work without an explicit path.

### 16.2 Asyncer Examples

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
