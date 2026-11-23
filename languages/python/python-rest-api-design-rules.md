---
paths:
  - "**/*.py"
  - "**/*.ipynb"
---

# REST API Design Rules — Python / FastAPI

When generating API endpoints, follow these conventions:

## URL Design
- Plural nouns for resources: `/users`, `/posts`, `/orders`
- Hierarchical: `/users/{id}/orders/{order_id}`
- Lowercase, kebab-case in the URL: `/blog-posts` not `/blogPosts`
- Version via `APIRouter(prefix="/api/v1")`, not a query param
- Actions as POST endpoints: `/users/{id}/activate`

## HTTP Methods
- GET: read (idempotent, cacheable)
- POST: create or action
- PUT: full replace
- PATCH: partial update
- DELETE: remove

## Status Codes
- 200 OK, 201 Created, 204 No Content
- 400 Bad Request (validation), 401 Unauthorized, 403 Forbidden, 404 Not Found
- 409 Conflict, 422 Unprocessable Entity (FastAPI's default for Pydantic validation failures)
- 429 Too Many Requests, 500 Internal Server Error

## Request/Response
- Pydantic models for every request body and `response_model` for every response — never return raw dicts/ORM objects
- Internal code stays snake_case; use `Field(alias=...)` or `alias_generator=to_camel` with `populate_by_name=True` on the model if the API contract requires camelCase JSON
- Raise `HTTPException(status_code=..., detail=...)` for expected error cases; let unexpected exceptions propagate to a global exception handler
- Consistent error shape via a registered `exception_handler`: `{ "error": { "code": ..., "message": ..., "details": ... } }`
- Pagination via `Query` params (`limit`, `offset` or cursor) with response shape `{ "data": [...], "meta": { "total": ..., "limit": ..., "offset": ... } }`

## Validation
- Use `Query`/`Path`/`Body` validators (`gt`, `max_length`, regex, etc.) at the parameter level instead of re-validating manually in the handler
- Keep request/response schemas separate from ORM/DB models (`UserCreate`/`UserResponse` vs. `User` table model) so the API contract doesn't leak internal schema

## Security
- Authenticate every non-public endpoint via a FastAPI `Depends` security dependency (OAuth2/JWT)
- Rate limiting on all endpoints (e.g. slowapi)
- Request size limits on file/body uploads
- Never return passwords, hashes, or secrets in a `response_model` — exclude them at the schema level, not by remembering to strip them in the handler

## Documentation
- FastAPI generates the OpenAPI spec automatically from routes/models — keep docstrings and `response_model`s accurate so `/docs` stays trustworthy; don't hand-maintain a separate spec
