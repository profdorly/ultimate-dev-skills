# Postgres: lock-safe schema changes

## Why locks matter

Most `ALTER TABLE` forms take an `ACCESS EXCLUSIVE` lock, which blocks reads and writes on the table. The lock itself is often brief, but **waiting** for it is the danger: while the `ALTER` waits behind a long-running transaction, every new query on that table queues behind the `ALTER`. A one-millisecond change can take the application down for as long as the oldest open transaction runs.

| Lock | Taken by | Blocks |
|---|---|---|
| `ACCESS EXCLUSIVE` | most `ALTER TABLE` forms (add, drop, or rename a column, change a type, `SET NOT NULL`), `DROP TABLE`, `TRUNCATE`, `REINDEX` (non-concurrent) | everything, including `SELECT` |
| `SHARE ROW EXCLUSIVE` | adding a foreign key (on both tables), `CREATE TRIGGER` | writes |
| `SHARE` | `CREATE INDEX` (non-concurrent) | writes, for the whole build |
| `SHARE UPDATE EXCLUSIVE` | `CREATE INDEX CONCURRENTLY`, `VALIDATE CONSTRAINT`, `VACUUM`, `ANALYZE` | other schema changes only; reads and writes continue |

**Always set a lock timeout** at the start of a migration that alters a table with traffic, so it fails fast instead of queueing:

```sql
SET lock_timeout = '5s';
```

If it times out, find the blocker (queries below) and retry later. Many teams retry automatically a few times with a pause. Linters catch most unsafe patterns before they run: `squawk` for SQL files, `strong_migrations` for Rails, `django-migration-linter` for Django. Use them if the project has them.

## Before you migrate

```sql
-- Approximate size of the table
SELECT pg_size_pretty(pg_total_relation_size('invoices')) AS total_size,
       (SELECT reltuples::bigint FROM pg_class WHERE oid = 'invoices'::regclass) AS approx_rows;

-- Long-running transactions that would make an ALTER wait
SELECT pid, now() - xact_start AS duration, state, left(query, 80) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL AND pid <> pg_backend_pid()
ORDER BY duration DESC
LIMIT 10;

-- Who is blocking whom right now
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event_type, left(query, 80) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

These are read-only, but run them against anything other than a local database only with the user's approval.

## Adding a column

| Change | Cost | Safe approach |
|---|---|---|
| Nullable, no default | Metadata only | Fine as is |
| With a constant or stable default (`'pending'`, `0`, `now()`), including `NOT NULL DEFAULT <constant>` | Metadata only (Postgres 11+) | Fine as is |
| With a volatile default (`gen_random_uuid()`, `random()`, `clock_timestamp()`) | Rewrites the table | Add it without a default; `ALTER TABLE t ALTER COLUMN c SET DEFAULT gen_random_uuid();` (affects new rows only); backfill existing rows in batches |
| A stored generated column | Rewrites the table | Treat like a volatile default, or schedule the rewrite with the user |

## Making a column NOT NULL

`ALTER COLUMN ... SET NOT NULL` scans the whole table under `ACCESS EXCLUSIVE`. Avoid the scan:

Postgres 18 and later:

```sql
ALTER TABLE invoices ADD CONSTRAINT invoices_due_date_not_null NOT NULL due_date NOT VALID;
-- backfill any NULLs, then, in a separate step:
ALTER TABLE invoices VALIDATE CONSTRAINT invoices_due_date_not_null;
```

Postgres 12 to 17:

```sql
ALTER TABLE invoices ADD CONSTRAINT invoices_due_date_not_null
  CHECK (due_date IS NOT NULL) NOT VALID;                        -- brief lock, no scan
ALTER TABLE invoices VALIDATE CONSTRAINT invoices_due_date_not_null;  -- scans, but allows reads and writes
ALTER TABLE invoices ALTER COLUMN due_date SET NOT NULL;           -- uses the valid check, skips the scan
ALTER TABLE invoices DROP CONSTRAINT invoices_due_date_not_null;
```

Run the `VALIDATE` in its own transaction (its own migration if the tool wraps each migration in a transaction), so the brief lock of the first statement isn't held during the scan.

## Indexes

```sql
CREATE INDEX CONCURRENTLY invoices_customer_id_idx ON invoices (customer_id);
```

- `CONCURRENTLY` builds without blocking writes, but it **can't run inside a transaction block**, and most migration tools wrap each migration in one. Turn that off for this migration only: Django `atomic = False` on the `Migration` class (and `AddIndexConcurrently` from `django.contrib.postgres.operations`); Rails `disable_ddl_transaction!` with `algorithm: :concurrently`; Alembic `with op.get_context().autocommit_block():` around `op.create_index(..., postgresql_concurrently=True)`; Knex `exports.config = { transaction: false }`. For other tools, check their docs for the per-migration setting.
- If a concurrent build fails or is cancelled, it leaves an **invalid** index behind. Find it, drop it, and retry:

  ```sql
  SELECT indexrelid::regclass AS index FROM pg_index WHERE NOT indisvalid;
  DROP INDEX CONCURRENTLY invoices_customer_id_idx;
  ```

  `CREATE INDEX CONCURRENTLY IF NOT EXISTS` silently skips a leftover invalid index, so check validity after building.
- **Unique constraints:** build the unique index concurrently, then attach it (brief lock):

  ```sql
  CREATE UNIQUE INDEX CONCURRENTLY users_email_key_idx ON users (email);
  ALTER TABLE users ADD CONSTRAINT users_email_key UNIQUE USING INDEX users_email_key_idx;
  ```

  The same `USING INDEX` form works for primary keys.
- Drop with `DROP INDEX CONCURRENTLY`. Rebuild with `REINDEX INDEX CONCURRENTLY` (Postgres 12+).
- On partitioned tables, a concurrent build on the parent isn't supported; create the index on each partition concurrently, then create the parent index with `ON ONLY` and attach the partition indexes.

## Foreign keys and check constraints

```sql
ALTER TABLE invoices ADD CONSTRAINT invoices_customer_id_fkey
  FOREIGN KEY (customer_id) REFERENCES customers (id) NOT VALID;   -- brief lock, no scan
ALTER TABLE invoices VALIDATE CONSTRAINT invoices_customer_id_fkey; -- scans without blocking writes
```

- Check constraints follow the same `NOT VALID` then `VALIDATE` pattern.
- Postgres doesn't index the referencing column automatically. Add the index (concurrently) so deletes on the parent and joins don't scan the child table.
- Fix violating rows before `VALIDATE`; it fails on the first violation.

## Changing a column's type

Most type changes rewrite the table under `ACCESS EXCLUSIVE`. Changes that don't rewrite:

- `varchar(n)` to a larger `varchar(m)`, to unconstrained `varchar`, or to `text`;
- removing a length limit;
- increasing a `numeric` column's precision with the same scale;
- `timestamp` to `timestamptz` (and back) when the session `TimeZone` is UTC (Postgres 12+).

Everything else, including `integer` to `bigint`, goes through expand and contract: add a new column, write to both, backfill in batches, switch reads, swap or drop. Ask before planning a rewrite of a large table; it may need a maintenance window.

## Renaming and dropping

- `RENAME COLUMN` and `RENAME TABLE` are instant but break code still using the old names. Use expand and contract, or for tables, rename and leave a view under the old name during the transition.
- `DROP COLUMN` is instant (the column is marked dropped; space is reclaimed later), but code that still selects it fails. Remove it from the code first and deploy; drop it in a later release. Rails caches column lists, so add the column to `ignored_columns` before the drop.

## Enums

- `ALTER TYPE invoice_status ADD VALUE 'archived';` is fast. Since Postgres 12 it can run inside a transaction, but the new value can't be used until that transaction commits, so don't insert rows with it in the same migration.
- Renaming a value: `ALTER TYPE ... RENAME VALUE` (Postgres 10+). Removing a value isn't supported; it needs a new type and a column migration. A `text` column with a `CHECK` constraint is easier to evolve.

## Backfills

Batch by primary key and commit each batch:

```sql
UPDATE invoices
SET due_date = issued_at + interval '30 days'
WHERE id IN (
  SELECT id FROM invoices
  WHERE due_date IS NULL
  ORDER BY id
  LIMIT 5000
);
```

Repeat until it updates zero rows, each batch in its own transaction, from a script or a job rather than one migration. Set a `statement_timeout` for each batch, pause between batches if replicas lag, and log progress. The `WHERE due_date IS NULL` condition makes reruns safe, but only if every batch actually fills the column: here a row with a NULL `issued_at` would stay NULL and be selected again forever. Handle such rows explicitly (a fallback value, or `AND issued_at IS NOT NULL` plus a separate decision about the rest), and stop the loop if a batch updates the same rows twice.

## Reading a plan without changing data

`EXPLAIN` shows the plan without running the statement. `EXPLAIN ANALYZE` **runs** it; for an `UPDATE` or `DELETE`, wrap it in a transaction you roll back:

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE invoices SET due_date = issued_at WHERE id < 1000;
ROLLBACK;
```

Even rolled back, it takes locks and does the work, so don't do this on production without approval.
