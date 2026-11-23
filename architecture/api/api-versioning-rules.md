---
paths:
  - "**/api/**"
  - "**/routes/**"
  - "**/openapi*.{yaml,yml,json}"
---

# API Versioning Rules

## Default scheme
- URL versioning is the default: `/api/v1/users` — visible, cacheable, and easy to route at the gateway/load-balancer level
- Use header versioning (`Accept: application/vnd.api+json;version=2`) only for content-negotiation-heavy APIs where clients need to opt into a version without changing the URL
- Avoid query-param versioning (`?version=1`) — easy to omit accidentally, breaks caching semantics

## Bumping versions
- Bump the major version only for breaking changes (removed fields, changed types, changed auth); additive changes (new optional fields, new endpoints) never require a version bump
- Support at most 2 major versions concurrently; forcing wider fan-out multiplies test/maintenance surface

## Deprecation & migration
- Every deprecated version must have a stated sunset date, communicated via a `Deprecation` and `Sunset` response header (RFC 8594)
- Publish a migration guide alongside the new version showing old-endpoint → new-endpoint mapping
- Log usage of deprecated versions so you know when it's safe to remove them, not just when the sunset date arrives
