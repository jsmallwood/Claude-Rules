---
paths:
  - "**/*.sql"
  - "**/migrations/**"
---

# SQL Rules

- Use snake_case for all table and column names; plural table names
- Always add created_at and updated_at timestamps to all tables
- Use UUIDs (uuid_generate_v4) or nanoid for primary keys in distributed systems
- Write queries that select only needed columns; never SELECT * in production
- Use materialized views for expensive aggregations that are read frequently
- Index foreign key columns automatically; analyze query plans for slow queries
- Use check constraints at the database level for enum-like values
- Test migrations on a copy of production data before running in production
- Always use parameterized queries; never concatenate user input into SQL strings — see rules-security.md
- Add indexes on foreign keys and columns used in WHERE, JOIN, ORDER BY clauses
- Use EXPLAIN ANALYZE to validate query performance before shipping
- Prefer CTEs (WITH) over nested subqueries for readability
- Use transactions for multi-step mutations; include rollback logic
- Add NOT NULL constraints and defaults at the database level
- Use RETURNING clause to avoid extra SELECT after INSERT/UPDATE
- Write backward-compatible migrations; never drop columns in the same deploy
