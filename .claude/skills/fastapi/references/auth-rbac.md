# Authentication, RBAC & Scope

> **Read when:** adding a permission, gating an endpoint, seeding roles, or working with scoped resources.

## 1. Two-Layer Access Control

Access control is split into **two independent layers**:

- **Permission layer** — `require_permission("resource:action")` gates whether
  a user can call an endpoint at all.
- **Scope layer** — `Depends(get_request_scope([...]))` returns a `ScopeContext`
  that filters the rows the endpoint returns/touches (own / team / branch / all).

Every scoped resource has a paired `<resource>:read_all` override permission.
The 13 baseline roles (super_admin → employee) are seeded by `app/user/seed.py`.

When adding a new scoped resource, add its permissions to `app/user/seed.py`
and wire the scope guard in the route using `get_request_scope`.

## 2. Permission Naming

```
<resource>:<action>          →  "leads:view"
<resource>:<action>:<scope>  →  "leads:view:all", "leads:view:team"
```

```python
# app/user/seed.py (or app/<feature>/permissions.py)
PERMISSIONS: list[tuple[str, str, str]] = [
    # (resource, action, description)
    ("leads", "view",     "View lead details"),
    ("leads", "view:all", "View all leads in the system"),
    ("users", "create",   "Create new user accounts"),
]
# Permission name stored in DB: "leads:view", "leads:view:all"
```

## 3. Role Definition

- Roles defined as constants in `user/seed.py`.
- `is_system=True` → cannot be deleted via UI.
- `super_admin` role bypasses ALL permission checks in the system.

## 4. Seed Script (`user/seed.py`)

- **Idempotent** — safe to run multiple times; only inserts missing data.
- Runs automatically on application startup (`lifespan` in `main.py`).
- Can also run manually: `python -m app.user.seed`.
- Use `dispose_engine=False` when called at startup (engine shared with app).
- Keep project-specific roles/permissions ONLY in seed.py — not scattered.

## 5. Gating an Endpoint

Three flavors, in order of preference:

```python
# Per-route guard
@router.get("/leads/{lead_id}")
async def get_lead(
    _: Annotated[None, Depends(require_permission("leads:view"))],
    lead_id: UUID,
    session: SessionDep,
) -> LeadResponse: ...

# Whole-router guard (when EVERY route needs it — see routers.md)
admin_router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_permission("admin:access"))],
)

# Scope filter for list endpoints (see list-endpoints.md)
async def list_leads(
    params: Annotated[LeadListParams, Depends()],
    session: SessionDep,
    current_user: CurrentUserDep,
) -> LeadListResponse:
    owner_user_ids = None
    if not current_user.has_permission("leads:read_all"):
        owner_user_ids = [current_user.id]
    return await lead_service.list_leads(session, params, owner_user_ids)
```
