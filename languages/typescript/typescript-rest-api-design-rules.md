---
paths:
  - "**/*.ts"
  - "**/*.tsx"
---

# REST API Design Rules (Typescript)

When generating API endpoints, follow these conventions:

## URL Design
- Plural nouns for resources: `/users`, `/posts`, `/orders`
- Hierarchical: `/users/{id}/orders/{orderId}`
- Lowercase, kebab-case: `/blog-posts` not `/blogPosts`
- Versions in URL: `/api/v1/users`
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
- 409 Conflict, 422 Unprocessable Entity
- 429 Too Many Requests, 500 Internal Server Error

## Request/Response
- JSON everywhere — consistent Content-Type: application/json
- camelCase field names in JSON
- Consistent error format: `{ error: { code, message, details } }`
- Pagination: `{ data: [], meta: { total, page, perPage, lastPage } }`

## Security
- Authenticate every non-public endpoint
- Rate limiting on all endpoints
- Request size limits
- Never return passwords or secrets in responses

## Documentation
- Generate and maintain an OpenAPI spec for every endpoint; keep it in sync with the implementation in CI