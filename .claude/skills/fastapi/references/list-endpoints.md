# List Endpoints — Filtering, Sorting & Pagination

> **Read when:** building any list endpoint that exposes filters, sort, and a total row count.

Every such endpoint **must** follow this standard. Companion deep-dive lives in
`app/core/crud/README.md` (full recipe) and `app/core/crud/SEARCH_STRATEGY.md`
(indexing & search strategy).

## The five-layer flow

```
Route (XxxListParams = Depends())
  └─ Service (thin pass-through + any business-rule overrides)
       └─ CRUD.get_list_filtered(session, skip, limit, sort_by, sort_order, …filters)
            └─ paginated_select(session, base_query, skip, limit, order_clauses)
                 └─ SQL: SELECT …, COUNT(*) OVER () OFFSET … LIMIT …
                 └─ Returns (items: list, total: int)
```

## Required pieces per module

| File | What to add |
|---|---|
| `enums.py` | `XxxSortField(str, Enum)` — the allowed sort columns |
| `schemas/xxx_schemas.py` | `XxxListParams(ListParams)` — override `sort_by` + add filter fields |
| `crud/xxx_crud.py` | `get_list_filtered(…)` — builds filters, calls `paginated_select()` |
| `services/xxx_service.py` | `list_xxx(session, params, owner_user_ids)` — pass-through + rules |
| `routes/xxx_routes.py` | `params: XxxListParams = Depends()` — scope injection |

## `XxxListParams` schema

```python
from app.core.schemas import ListParams
from app.<feature>.enums import ThingSortField, SortDirection

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

## Scope enforcement in routes

Pydantic models are immutable — never mutate `params` fields. Use a local variable:

```python
from typing import Annotated

@router.get("/")
async def list_things(
    params: Annotated[ThingListParams, Depends()],
    session: SessionDep,
    current_user: CurrentUserDep,
    thing_service: ThingServiceDep,
) -> ThingListResponse:
    owner_user_ids = None
    if not current_user.has_permission("things:read_all"):
        owner_user_ids = [current_user.id]   # ← local var, not params mutation

    return await thing_service.list_things(session, params, owner_user_ids)
```

## `paginated_select()` — single-query pagination

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

## Sort column tiebreaker

Always append `model.id.desc()` as the last element of `order_clauses`.
Without it, `OFFSET`/`LIMIT` can return duplicate or skipped rows when rows
are inserted between page requests on a non-unique sort key.
