# Adding a New Feature Module

> **Read when:** scaffolding a new `app/<feature>/` from scratch.

This is the master checklist. For each step, load the named reference file from
the SKILL.md routing map if you need full details.

## Steps

1. **Create the module directory.** Standard sub-structure (`models/`, `schemas/`,
   `crud/`, `services/`, `routes/`, `dependencies.py`, `exceptions.py`, `enums.py`,
   `permissions.py`, `tasks.py`). Full layout in `structure.md`.

2. **Define models.** UUID primary key, `DateTime(timezone=True)`, `default=utc_now`,
   soft-delete columns, named indexes. Conventions in `database.md`.

3. **Add models to Alembic import file.**
   Edit `app/core/alembic_models_import.py` and import every model class.
   Without this, autogenerate won't see them.

4. **Define Pydantic schemas.** `XxxCreate`, `XxxUpdate`, `XxxResponse`.
   Use `Annotated` + `Field`, no Ellipsis, no `RootModel`. See `request-response.md`.

5. **Write CRUD class.** `class XxxCRUD(CRUDBase[Xxx, XxxCreate, XxxUpdate])`.
   No commits inside CRUD — only `flush` / `refresh`. See `architecture.md`.

6. **Write service class.** Constructor takes CRUD as argument; service owns
   `session.commit()`. See `architecture.md`.

7. **Wire DI in `dependencies.py`.** `@lru_cache` factories for both CRUD and
   service, plus `Annotated` `*Dep` aliases. See `dependency-injection.md`.

8. **Add permissions.** Edit `app/user/seed.py` (`PERMISSIONS` +
   `ROLE_PERMISSIONS`). See `auth-rbac.md`.

9. **Build the router.** `APIRouter(prefix="/xxx", tags=["xxx"])` with
   `dependencies=[Depends(require_permission(...))]` if every route shares a
   guard. See `routers.md`.

10. **Register in `app/apis/v1.py`.** `router.include_router(xxx_router)` —
    no prefix/tags here.

11. **For any list endpoint:** define `XxxSortField` enum →
    `XxxListParams(ListParams)` schema → `get_list_filtered()` CRUD method →
    service pass-through → route with `Depends()`. Full recipe in `list-endpoints.md`.

12. **Generate & apply migration.**
    ```bash
    alembic revision --autogenerate -m "add <feature> models"
    # Review the generated file in migrations/versions/
    alembic upgrade head
    ```

13. **Background tasks (if any):** `app/<feature>/tasks.py`. See `celery.md`.

14. **Module-specific exceptions:** `app/<feature>/exceptions.py` —
    `HTTPException` subclasses raised from services. See `operations.md`.

## Skill Defaults Overridden by This Project

| Topic | Upstream default | This project |
|---|---|---|
| SQL ORM | SQLModel | Async SQLAlchemy 2.x + `CRUDBase[Model, Create, Update]` |
| Route style | `def` when in doubt | `async def` always |
| CRUD/service wiring | Module-level singletons | `@lru_cache` factories in `dependencies.py` |
| HTTP client | (HTTPX recommended) | HTTPX — adopted; no `requests` |
| Async/blocking bridge | (Asyncer recommended) | Asyncer — adopted; no `anyio.to_thread` |

Everything else from the upstream FastAPI skill applies as written.
