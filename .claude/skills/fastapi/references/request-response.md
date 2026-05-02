# Request Parameters, Responses & Streaming

> **Read when:** writing any route signature, defining response models, or building a streaming endpoint.

## 1. FastAPI Parameter Declarations

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

**Never use Ellipsis (`...`) for required fields.** A field without a default *is* required.

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
@router.post("/items/")
async def create_items(items: Annotated[list[int], Field(min_length=1), Body()]):
    return items
```

**One HTTP method per function.** Never use `@router.api_route(methods=[...])`.
Split into `@router.get(...)` + `@router.post(...)` etc.

## 2. Response Handling

**Prefer return-type annotation over `response_model=`.** The return type drives
validation, filtering, and serialization (Pydantic-Rust path).

```python
# ✅ Preferred
@router.get("/me")
async def get_me(current_user: CurrentUserDep) -> UserResponse:
    return UserResponse.model_validate(current_user)
```

Use `response_model=` **only** when the actual returned type and the documented
type genuinely differ:

```python
@router.get("/me", response_model=UserPublic)
async def get_me(current_user: CurrentUserDep) -> Any:
    return current_user   # InternalUser → filtered down to UserPublic
```

**Do NOT use `ORJSONResponse` or `UJSONResponse`** — both are deprecated.
Pydantic v2 already serializes via Rust at native speed.

## 3. Streaming

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
`response_class=`, and `yield` / `yield from` from the function. Do not return
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
