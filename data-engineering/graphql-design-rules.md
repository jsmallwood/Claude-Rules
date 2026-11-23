---
paths:
  - "**/*.graphql"
  - "**/*.gql"
  - "**/schema/**"
  - "**/resolvers/**"
---

# GraphQL Design Rules

## Schema design
- Design schema-first: write the SDL before implementation so the contract is reviewable independent of resolver code
- Name types and fields for the domain, not the database — `User.displayName`, not `User.display_name_col`
- Prefer nullable fields over non-null unless the field is truly always present; a non-null field that can fail nulls the whole parent object on error

## Resolvers & performance
- Batch and cache with DataLoader (or equivalent) for every field that fetches by ID — prevents N+1 queries when a list field is requested
- Keep resolvers thin; push business logic into services so it's testable outside the GraphQL layer
- Apply query depth limiting and query complexity/cost analysis to reject expensive queries before execution

## Pagination
- Use cursor-based (Relay-style `edges`/`node`/`pageInfo`) pagination for any list that can grow unbounded; offset pagination only for small, stable lists
- Always cap `first`/`last` with a max page size enforced server-side

## Authentication & authorization
- Authenticate once at the request/context level; pass the resolved identity through resolver context rather than re-authenticating per field
- Authorize per-field for sensitive data (e.g. `User.email` visible only to self/admin) rather than only at the query root

## Security (Apollo / production hardening)
- Disable introspection and the GraphQL Playground/sandbox in production
- Use persisted queries (automatic persisted queries or a fixed allowlist) in production to prevent arbitrary/expensive ad-hoc queries from clients
- Apply the same rate limiting and input validation as any other API — see rules-security.md; a single `/graphql` endpoint doesn't get a pass on REST-equivalent controls
