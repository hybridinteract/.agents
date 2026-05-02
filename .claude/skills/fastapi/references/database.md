# Database, Alembic & Datetime

> **Read when:** adding a model, writing/running a migration, or working with timestamps.

## 1. Model Conventions

```python
class MyModel(Base):
    __tablename__ = "plural_snake_case"
    __table_args__ = (
        Index('ix_mytable_field', 'field'),   # Always name indexes explicitly
    )

    # Primary key — always UUID
    id: Mapped[UUIDType] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid4
    )

    # Timestamps — always timezone=True, always UTC
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utc_now, nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utc_now, onupdate=utc_now, nullable=False
    )

    # Soft delete pattern
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    deleted_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
```

## 2. Alembic Model Import (Single Source of Truth)

`app/core/alembic_models_import.py` — the single file Alembic autogenerate scans:

```python
# Import Base for metadata
from app.core.models import Base

# User module
from app.user.models import (
    User, Role, Permission, RefreshToken, UserRole, RolePermission,
)

# Add every new module here ↓
from app.activity.models import ActivityLog
from app.release_notes.models import ReleaseNote
# from app.<feature>.models import MyModel
```

`migrations/env.py` does `from app.core.alembic_models_import import *` — add
models only to the import file, never directly to env.py.

## 3. Migration File Naming

```
YYYY_MM_DD_HHMM-<rev>_<description>.py
2026_02_09_0657-8600ba4ec5f7_user_init_models.py
```

Configured in `alembic.ini`:

```ini
file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s
```

## 4. Datetime Standards

- **All datetime fields**: `DateTime(timezone=True)` — stores with timezone.
- **Default**: `default=utc_now` (from `app.core.utils`).
- **Never** call `datetime.utcnow()` — deprecated. Use `datetime.now(timezone.utc)`.
- **Frontend handles display timezone conversion** — backend always stores UTC.

## 5. Workflow

```bash
# 1. Edit model in app/<feature>/models/
# 2. Add the model class to app/core/alembic_models_import.py
# 3. Generate migration (runs inside the api container)
docker compose exec api alembic revision --autogenerate -m "<description>"
# 4. Review the generated file in migrations/versions/
# 5. Apply
docker compose exec api alembic upgrade head
```
