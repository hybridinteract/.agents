# Settings & Configuration

> **Read when:** adding a config value, reading a secret, or editing `app/core/settings.py` / `.env.example`.

## 1. Pydantic Settings Class (`app/core/settings.py`)

- Use `BaseSettings` with `.env` file support.
- Group fields with section comments:
  ```python
  # ==================== Database Settings ====================
  ```
- Computed values (built from other fields) use `@property`.
- Required secrets have **no default** — they fail fast on startup:
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
- Cache settings with `@lru_cache()` — only one instance per process.

## 2. Environment Variables

All env vars must be documented in `.env.example`:

```bash
# ==================== Section Name ====================
VAR_NAME=default_value     # Inline comment explaining purpose
```

Never commit a real `.env`; always update `.env.example` when adding a setting.
