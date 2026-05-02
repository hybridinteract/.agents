---
name: fastapi
description: >
  Provides FastAPI conventions for this backend codebase. Use when writing or
  refactoring routes, services, CRUDs, Pydantic schemas, SQLAlchemy models,
  Alembic migrations, or wiring dependencies. Covers async SQLAlchemy 2.x +
  CRUDBase, @lru_cache DI factories, RBAC scopes, and Celery tasks.
when_to_use: >
  Triggered when working on files under app/ or migrations/, or when the user
  mentions FastAPI, SQLAlchemy, Alembic, Pydantic, CRUD, service layer,
  dependencies.py, or module scaffolding.
paths: "app/**/*.py, migrations/**/*.py"
allowed-tools: "Bash(alembic *) Bash(fastapi *) Bash(uv *) Bash(ruff *)"
argument-hint: "[module-name or topic]"
---

# FastAPI — Backend

This skill is a **task-aware index** to the project's FastAPI conventions.
Detailed conventions are split across focused reference files in `references/`.
**Load only the references relevant to the current task** to keep the context
window small.

---

## Non-Negotiables (always in effect)

These rules apply to every change, regardless of which reference file you load.
Each links to its full treatment.

| Rule | Reference |
|---|---|
| **Always `async def`** for routes, services, CRUDs | [architecture.md](references/architecture.md) |
| Strict layered architecture: route → service → CRUD → model | [architecture.md](references/architecture.md) |
| Service owns `session.commit()`; CRUD only `flush` / `refresh` | [architecture.md](references/architecture.md) |
| **No CRUD/service singletons** — `@lru_cache` factories in `dependencies.py` | [dependency-injection.md](references/dependency-injection.md) |
| Always `Annotated` for `Path`/`Query`/`Header`/`Body` and dependencies | [request-response.md](references/request-response.md) |
| No Ellipsis (`...`) defaults; no Pydantic `RootModel`; one HTTP method per function | [request-response.md](references/request-response.md) |
| Return-type annotation preferred over `response_model=` | [request-response.md](references/request-response.md) |
| Async SQLAlchemy 2.x with `Mapped[]` + `mapped_column` — **never SQLModel** | [database.md](references/database.md) |
| `DateTime(timezone=True)` + `default=utc_now`; never `datetime.utcnow()` | [database.md](references/database.md) |
| Permission strings: `<resource>:<action>[:<scope>]` | [auth-rbac.md](references/auth-rbac.md) |
| New model → also add to `app/core/alembic_models_import.py` | [database.md](references/database.md) |
| HTTPX (no `requests`); Asyncer (no `anyio.to_thread`) | [tooling.md](references/tooling.md) |

---

## Routing Map — Task → Reference File

Match the task in front of you against the left column and load only that file.

| You are working on… | Load |
|---|---|
| Scaffolding a brand-new module | [new-module.md](references/new-module.md) |
| Project layout / where a file belongs / naming | [structure.md](references/structure.md) |
| Routes, services, CRUDs (architecture, transactions) | [architecture.md](references/architecture.md) |
| List endpoint with filter / sort / pagination | [list-endpoints.md](references/list-endpoints.md) |
| Path / Query / Header / Body parameters, response models, streaming | [request-response.md](references/request-response.md) |
| Registering a router or hoisting shared dependencies | [routers.md](references/routers.md) |
| Wiring a CRUD/service factory; `yield` cleanup; class-deps | [dependency-injection.md](references/dependency-injection.md) |
| Models, Alembic migrations, datetime fields | [database.md](references/database.md) |
| Adding a setting / env var | [settings.md](references/settings.md) |
| Permissions, roles, scope, seeding | [auth-rbac.md](references/auth-rbac.md) |
| Background / Celery task | [celery.md](references/celery.md) |
| Exceptions, logging, Docker | [operations.md](references/operations.md) |
| `fastapi` CLI, uv, Ruff, ty, HTTPX, Asyncer | [tooling.md](references/tooling.md) |

When in doubt, start with **new-module.md** — it links out to every other file
in checklist order.

---

## Skill Overrides at a Glance

The upstream FastAPI skill ships with a few defaults that **do not apply here**.

| Topic | Upstream default | This project |
|---|---|---|
| SQL ORM | SQLModel | Async SQLAlchemy 2.x + `CRUDBase[Model, Create, Update]` |
| Route style | `def` when in doubt | `async def` always |
| CRUD/service wiring | Module-level singletons | `@lru_cache` factories in `dependencies.py` |
| HTTP client | (HTTPX recommended) | HTTPX — adopted; no `requests` |
| Async/blocking bridge | (Asyncer recommended) | Asyncer — adopted; no `anyio.to_thread` |

Everything else from the upstream FastAPI skill applies as written.

---

## CLI

```bash
# Start only the API (hot-reload via uvicorn --reload inside the container)
docker compose up api

# Start the full stack (postgres, redis, api, celery_worker, flower)
docker compose up

# Rebuild after dependency changes
docker compose up --build api

# Alembic migrations (runs inside the api container)
docker compose exec api alembic revision --autogenerate -m "<description>"
docker compose exec api alembic upgrade head
```

The `api` container mounts `./app` read-only and passes `--reload-dir /app/app`
to uvicorn, so code changes reflect immediately without a rebuild.
