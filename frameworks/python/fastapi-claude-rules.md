---
paths:
  - "**/*.py"
---

- Use async SQLAlchemy with asyncpg for database access
- Structure with: routers/, schemas/, services/, repositories/ directories
- Use alembic for migrations; never use SQLAlchemy's create_all in production
- Write pytest-asyncio tests; use HTTPX AsyncClient for testing endpoints
- Add structlog for structured JSON logging with request correlation IDs
- Use slowapi for rate limiting; implement it at the router level
- Use Redis + fastapi-cache2 for response caching with per-endpoint TTLs
- Enable OpenTelemetry tracing for distributed observability