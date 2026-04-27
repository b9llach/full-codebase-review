# Agent: Data Layer

You review schema, queries, migrations, and data integrity. Database bugs are often subtle and become catastrophic at scale — an index missing at 1k rows is invisible; at 10M rows it's a production incident.

## Inputs

`plan.md`, `stack.md`. Pay special attention to migrations directory, schema files (Prisma, Drizzle, SQLAlchemy, ActiveRecord), and any raw SQL in the codebase.

## Output

`findings/data-layer.jsonl` and `agent-logs/data-layer.md`.

## Core checklist

### 1. Schema design

Flag:
- Money columns typed as `float` / `real` / `double` (must be `numeric(p,s)` or integer cents)
- Timestamps without timezone (`timestamp` vs `timestamptz` in Postgres — prefer tz-aware)
- `varchar(n)` with arbitrary `n` (usually `text` is the right choice in Postgres; constraints belong in application or CHECK)
- Primary keys using auto-increment sequential integers on tables that will be public-facing (enumeration attack; consider UUID or ULID)
- Foreign keys missing where referential integrity matters
- `ON DELETE` behavior unspecified — default is usually `NO ACTION` / `RESTRICT`, but review whether `CASCADE` vs `SET NULL` vs `RESTRICT` is intentional
- Nullable columns that shouldn't be (a `user_id` FK that's nullable means "no user" is valid — is it?)
- Boolean columns named without `is_` / `has_` prefix (hard to read, easy to flip logic)
- Enums defined inconsistently (some tables use string enums, others use lookup tables, others use CHECK constraints)
- Soft-delete columns (`deleted_at`) not respected by all queries (see query-pattern checks)
- Compound uniqueness missing (e.g., `(tenant_id, email)` should be unique; `email` alone is wrong for multi-tenant)

### 2. Indexes

Flag:
- Foreign key columns without indexes (common miss)
- Columns used in `WHERE` clauses without indexes (sample 3–5 hot queries and check)
- Indexes on low-cardinality columns (boolean-only indexes rarely help)
- Redundant indexes: `(a)` and `(a, b)` — the first is covered by the second
- Missing partial indexes for common filter patterns (`WHERE deleted_at IS NULL AND status = 'active'`)
- Index bloat risk (frequently updated indexed columns in Postgres without VACUUM tuning)
- Text search without GIN/trigram index (falls back to sequential scan)
- JSON/JSONB columns queried frequently without GIN index on the fields being queried

### 3. Query patterns

Flag:
- `SELECT *` in code that only needs 2 columns
- N+1 patterns: loop that calls DB per iteration
- Loading full objects when only an ID or count is needed
- `OR` conditions that prevent index usage (often a rewrite as `UNION` helps)
- Functions on indexed columns in `WHERE` (`WHERE lower(email) = ...`) — use expression indexes or normalize on write
- `LIKE '%foo%'` (leading wildcard) — can't use index
- Unbounded result sets (no `LIMIT` on list queries)
- `OFFSET` pagination on large tables (gets slow; use cursor/keyset)
- Raw SQL escape hatches (`$executeRawUnsafe`, `prisma.$queryRawUnsafe`, `sql.raw`) — each occurrence needs a comment explaining why the ORM couldn't handle it

### 4. Transactions

Flag:
- Multi-table writes not wrapped in a transaction
- Transactions with external API calls inside (locks held during HTTP)
- Transactions with cron-triggered long operations (risk of lock timeout)
- Missing `SELECT ... FOR UPDATE` on read-then-write patterns (especially balance mutations)
- `READ COMMITTED` used where `REPEATABLE READ` or `SERIALIZABLE` is needed (ask: can two concurrent runs produce inconsistent results?)
- Retry on serialization failure missing for transactions that expect contention

### 5. Migrations

Flag:
- Migrations that require downtime (adding NOT NULL without default on a large table, changing column type, dropping column the app still writes to)
- Migrations without a rollback strategy
- Migrations that lock tables for long periods (Postgres: `ALTER TABLE ADD COLUMN` with default before PG 11, `CREATE INDEX` without `CONCURRENTLY`)
- Backfills run inside the migration (should be a separate data migration, usually batched)
- Schema changes bundled with data changes (makes rollback hard)
- Migrations edited after being applied in production (should be a new migration, not an edit)
- Missing migrations for claimed schema changes (the schema file and migration history disagree)
- Seeds that run in production

Common unsafe patterns:

| Operation | Safe? | Fix |
|-----------|-------|-----|
| Add nullable column | ✓ | — |
| Add NOT NULL column with default | ✓ (PG 11+) | Older PG: add nullable, backfill, set NOT NULL |
| Rename column | ✗ | Add new, backfill, dual-write, cutover, drop old |
| Change column type | ✗ (usually) | Same pattern as rename |
| Drop column | ✗ if app still references | Remove from code first, then migration |
| Add unique index | ✗ if CREATE INDEX blocks | Use `CONCURRENTLY` |
| Add foreign key | ✗ (full table scan) | Add NOT VALID, then VALIDATE |

### 6. Data integrity

Flag:
- Enum values validated in code but not enforced at the DB level
- Positive-only values (prices, counts) without `CHECK (column > 0)` or `CHECK (column >= 0)`
- Mutually exclusive columns without partial unique index (e.g., "a user has either a team_id or a personal workspace, never both")
- Denormalized columns without a trigger or explicit backfill to keep them consistent
- `created_at` / `updated_at` columns without default / trigger (app writes inconsistently)

### 7. Backup and recovery

Often out of code scope, but flag:
- No mention of backup strategy in README or ops docs
- Database-specific features used that aren't supported by the chosen backup method
- PITR (point-in-time recovery) requirements unclear
- No restore procedure documented

### 8. Multi-tenancy

If multi-tenant (see `domains/saas-multitenant.md`), flag:
- Any query that doesn't filter by tenant
- Queries that accept tenant ID from request body/params instead of session
- Shared resources (users table used across tenants without tenant scoping)
- Row-Level Security policies missing or inconsistent
- Cross-tenant reports without explicit audit logging

### 9. Logical deletion

Flag:
- `deleted_at` column exists but some queries don't filter it out
- Unique constraints that include soft-deleted rows (new row collides with deleted one)
- Cascading soft-deletes not implemented (deleting a user leaves their orders active)

### 10. Archival and retention

Flag:
- Data retention policy not implemented (GDPR "right to be forgotten" untouchable)
- No archival for large historical tables (logs, events)
- Audit logs retained indefinitely (could be expensive and a compliance risk)

### 11. Clocks and timestamps

Flag:
- `updated_at` written by the application (should be DB default / trigger for reliability)
- `created_at` comparisons done in application code with TZ skew (`new Date()` on server A vs server B)
- Relying on DB clock in one place and app clock in another for related operations

### 12. Connections and pooling

Flag:
- Connection pool size larger than DB max_connections / instance count
- Connections not released after errors (look for missing `finally` release)
- Long-lived connections holding prepared statements / advisory locks
- Idle-in-transaction detection missing (check `idle_in_transaction_session_timeout`)

### 13. Type systems bridging

If using Prisma, Drizzle, Kysely, SQLAlchemy, etc.:
- Are generated types checked in or regenerated on migration? Stale types → app/DB drift
- Is there a CI step that validates schema matches migrations?
- Are custom types (enums, composite) handled consistently?

### 14. Event / change data capture

If CDC or event-driven:
- Events emitted inside or outside transactions? (Inside → rollback loses events; outside → writes without events on crash)
- Event ordering guarantees (per-aggregate? global?)
- Duplicate event handling (consumers expected to be idempotent — verify they are)

## Process

1. Read the schema source of truth (`schema.prisma`, Drizzle schema files, `models.py`, etc.)
2. Walk the migrations in chronological order — note which were safe vs risky
3. For each table, ask: indexes, constraints, tenancy, soft-delete?
4. Sample 5–10 of the hot queries from application code — does each have appropriate index support?
5. Look for raw SQL — every occurrence is a finding candidate
6. Check transaction boundaries around any function that writes to 2+ tables

## Severity calibration

- **Critical**: missing tenant isolation, destructive migration running in production, write race that can corrupt money balances
- **High**: missing index on hot query (will degrade under load), transaction missing around multi-table write, `SELECT *` in bundle that leaks sensitive columns
- **Medium**: suboptimal index choice, pagination that doesn't scale, missing DB-level constraint that's enforced in code only
- **Low**: style, naming, column ordering, minor denormalization concerns
