---
paths:
  - "**/*.sql"
  - "**/migrations/**"
  - "**/models/**"
---

# Generic Database Rules

## Schema
- Every table has primary key, created_at, updated_at.
- Add indexes for foreign keys and frequent filters.
- Model constraints in DB (unique, foreign keys, check constraints).
- Prefer soft deletes (`deleted_at` timestamp) over hard deletes for user-facing/auditable data; hard-delete only where retention rules require it.
- Add audit fields (`created_by`, `updated_by`) on tables where "who changed this" matters for compliance or debugging.

## Migrations
- Forward-only migrations; never edit applied migration files.
- Pair risky migrations with rollback/runbook notes.
- Backfill large changes in chunks, not one giant lock-heavy operation.

## Query discipline
- Select only required columns.
- Use transactions for multi-step state changes.
- Use explain plans for slow queries before shipping.