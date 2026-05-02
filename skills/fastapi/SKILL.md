---
name: fastapi
description: FastAPI conventions for the Herbally backend. Use when writing or refactoring FastAPI routes, services, CRUDs, or Pydantic schemas in this project. Wraps the upstream FastAPI best practices and overlays project-specific overrides (async SQLAlchemy + CRUDBase, factory-based DI, RBAC + scope, etc.).
---

# FastAPI — Herbally Backend

> **Single source of truth:** [`references/PROJECT_CONVENTIONS.md`](references/PROJECT_CONVENTIONS.md)
>
> Always read it before writing FastAPI code in this project. It absorbs the
> upstream FastAPI skill and overlays the project-specific rules. When the
> conventions and the upstream framework guidance conflict, **the conventions win**.

---

## When to use this skill

Use it whenever you are:
- Adding a new feature module under `app/<feature>/`
- Adding or modifying a route, service, CRUD, or Pydantic schema
- Wiring a new dependency, permission, or background task
- Touching `app/apis/v1.py`, `app/core/`, or `migrations/`

---

## Read first, then code

1. Open [`references/PROJECT_CONVENTIONS.md`](references/PROJECT_CONVENTIONS.md).
2. Skim §0 (skill overrides) and §13 (new-module checklist).
3. Jump to the section relevant to your task — the sections are numbered and
   self-contained.

If you only have time for one section, read **§3 Architecture Patterns**.

---

## Non-negotiables (do not break)

These are the project rules that get violated most often. They are documented
in full in PROJECT_CONVENTIONS.md — listed here purely as a checklist.

| Rule | Where it lives |
|---|---|
| **Always `async def`** for routes, services, CRUDs | §3.1 |
| Layered architecture: route → service → CRUD → model (never skip) | §3.1 |
| Service owns `session.commit()`; CRUD only `flush`/`refresh` | §3.2 |
| **No CRUD/service singletons** — use `@lru_cache` factories in `dependencies.py` | §3.3, §15.1 |
| Always `Annotated` for `Path`/`Query`/`Header`/`Body` and dependencies | §3.6, §15 |
| No Ellipsis (`...`) defaults; no Pydantic `RootModel`; one HTTP method per function | §3.6 |
| Return-type annotation preferred over `response_model=` | §3.7 |
| Async SQLAlchemy 2.x with `Mapped[]` + `mapped_column` — **never SQLModel** | §5.1 |
| `DateTime(timezone=True)` + `default=utc_now`; never `datetime.utcnow()` | §14 |
| Permission strings: `<resource>:<action>[:<scope>]` | §6.1 |
| Add new model to `app/core/alembic_models_import.py` | §5.2 |

---

## Skill overrides at a glance

The upstream FastAPI skill ships with a few defaults that **do not apply here**.
Reproduced from PROJECT_CONVENTIONS §0 so you don't have to context-switch:

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
# Dev (auto-reload)
fastapi dev

# Production (Docker is the primary deploy target — see PROJECT_CONVENTIONS §11)
fastapi run
```

Entrypoint is read from `pyproject.toml`:

```toml
[tool.fastapi]
entrypoint = "app.core.main:app"
```

---

## Adding a new feature module — quick path

Full recipe in PROJECT_CONVENTIONS §13. The short version:

1. `app/<feature>/` with the standard sub-structure (§1)
2. Add models to `app/core/alembic_models_import.py` (§5.2)
3. Add permissions to `app/user/seed.py` (§6.1)
4. Wire CRUDs and services as `@lru_cache` factories in `app/<feature>/dependencies.py` (§15.1)
5. Register the router in `app/apis/v1.py` (§7)
6. `alembic revision --autogenerate -m "<description>"` → `alembic upgrade head`
7. For any list endpoint: define `XxxSortField` enum → `XxxListParams(ListParams)` → `get_list_filtered()` → service pass-through → route with `Depends()` (§3.5)

---

## Where to look

| You're working on… | Read this section of PROJECT_CONVENTIONS.md |
|---|---|
| Project layout, naming | §1, §2 |
| Routes, services, CRUDs (architecture) | §3.1 – §3.3 |
| List endpoints (filter/sort/pagination) | §3.5 |
| Path/Query/Header/Body parameters | §3.6 |
| Response models, return types | §3.7 |
| Streaming (JSONL, SSE, bytes) | §3.8 |
| Settings | §4 |
| Models, Alembic, migrations | §5 |
| Auth, RBAC, permissions | §6, §1.1 |
| Router registration | §7 |
| Celery tasks | §8 |
| Exceptions, logging | §9, §10 |
| Docker | §11 |
| Datetime | §14 |
| Dependency injection (factories, scopes, class deps) | §15 |
| Tooling (uv, Ruff, ty, fastapi CLI, HTTPX, Asyncer) | §16 |
