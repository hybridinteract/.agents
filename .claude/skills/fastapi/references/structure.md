# Project Structure & Naming

> **Read when:** scaffolding a new module, adding a file, or unsure where something belongs.

## Contents
- [1. Directory Layout](#1-directory-layout)
- [2. Naming Conventions](#2-naming-conventions) — files, Python identifiers, database, permissions
- [Decision Guide — What Goes Where](#decision-guide--what-goes-where)

## 1. Directory Layout

```
<project_name>/
├── app/
│   ├── __init__.py
│   ├── core/                        # ✅ Reusable — never project-specific
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
│   │   ├── crud/                    # CRUDBase, apply_sorting, paginated_select
│   │   ├── background/              # Celery app + task infrastructure
│   │   ├── cache/                   # Redis cache abstraction
│   │   └── object_storage/          # S3-compatible file storage
│   │
│   ├── apis/
│   │   └── v1.py                    # Aggregates all module routers
│   │
│   ├── user/                        # ✅ Reusable — auth + RBAC
│   ├── activity/                    # ✅ Reusable — append-only audit log
│   ├── release_notes/               # ✅ Reusable — What's New system
│   │
│   └── <feature>/                   # 🔧 Project-specific feature modules
│       ├── __init__.py              # Module public API
│       ├── models/                  # DB models
│       ├── schemas/                 # Pydantic schemas
│       ├── crud/                    # CRUD classes
│       ├── services/                # Business logic
│       ├── routes/                  # FastAPI routers
│       ├── dependencies.py          # FastAPI Depends() helpers + factories
│       ├── exceptions.py            # Module-specific exceptions
│       ├── enums.py                 # Module-specific enums
│       ├── permissions.py           # Permission constants for the module
│       └── tasks.py                 # Celery tasks for the module
│
├── migrations/                      # Alembic
├── docker/                          # Dockerfile + entrypoints
├── docs/                            # Module-level documentation
├── logs/                            # Runtime log files (gitignored)
├── alembic.ini
├── docker-compose.yml
├── pyproject.toml
├── .env.example
└── README.md
```

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
| FK constraint | SQLAlchemy default | (auto-named) |
| Association tables | `<table1>_<table2>` | `user_roles`, `role_permissions` |

### Permissions

```
<resource>:<action>          →  "leads:view"
<resource>:<action>:<scope>  →  "leads:view:all", "leads:view:team"
```

## Decision Guide — What Goes Where

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
