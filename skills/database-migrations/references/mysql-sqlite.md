# MySQL, MariaDB, and SQLite

## MySQL and MariaDB (InnoDB)

### Say which algorithm you expect

Add `ALGORITHM` and `LOCK` clauses to every `ALTER TABLE` on a table with traffic. If MySQL can't do the change that way, it fails immediately with an error instead of silently copying the whole table under a lock:

```sql
ALTER TABLE invoices ADD COLUMN due_date DATE NULL, ALGORITHM=INSTANT;
ALTER TABLE invoices ADD INDEX invoices_customer_id_idx (customer_id), ALGORITHM=INPLACE, LOCK=NONE;
```

| Change | Usually possible as |
|---|---|
| Add a column (MySQL 8.0.29+: at any position; 8.0.12 to 8.0.28: only as the last column) | `INSTANT` |
| Drop a column (MySQL 8.0.29+) | `INSTANT` |
| Add or drop a secondary index | `INPLACE, LOCK=NONE` |
| Rename a column or index | `INSTANT` or `INPLACE` |
| Extend a `VARCHAR` when the old and new sizes are both at most 255 bytes, or both at least 256 bytes (the length prefix keeps its size) | `INPLACE, LOCK=NONE` |
| Replace the primary key (drop and add in one statement) | `INPLACE, LOCK=NONE` (dropping it without adding a new one needs `COPY`) |
| Change a column's type (for example `INT` to `BIGINT`), convert the character set | `COPY` only: the table is rebuilt and writes are blocked |

MariaDB's support differs by version; check its documentation for the exact release.

### Metadata locks

Every `ALTER TABLE` needs a metadata lock, and while it waits behind an open transaction, new queries on that table queue behind it. `lock_wait_timeout` defaults to 31,536,000 seconds (one year), so set it for the migration session:

```sql
SET SESSION lock_wait_timeout = 5;
```

Find what's holding things up (read-only):

```sql
SELECT * FROM sys.schema_table_lock_waits;          -- needs the sys schema and performance_schema
SELECT trx_id, trx_started, trx_mysql_thread_id, LEFT(trx_query, 80)
FROM information_schema.innodb_trx ORDER BY trx_started;
```

### DDL is not transactional

MySQL commits implicitly around each DDL statement. A migration with several statements that fails halfway leaves the earlier ones applied, and the migration tool may record it as failed or not at all. So:

- put one DDL statement in each migration where you can;
- after a failure, inspect the real schema (`SHOW CREATE TABLE <name>`) before retrying or rolling back;
- write down migrations that tolerate partial application (check before dropping or adding).

### Large tables

For changes that need `COPY` on a large, busy table, online schema-change tools (`gh-ost`, `pt-online-schema-change`) copy the table in the background and swap it in. Choosing and running them is the user's or the DBA's decision; propose, don't run.

### Backups

```bash
mysqldump --single-transaction --routines --triggers --databases <db> > backup.sql
```

`--single-transaction` gives a consistent snapshot of InnoDB tables without locking them. Check the file (`head -n 30 backup.sql`) and keep it out of the repository.

## SQLite

### What `ALTER TABLE` can do

| Operation | Support |
|---|---|
| `RENAME TO` (table) | All versions |
| `RENAME COLUMN` | 3.25.0+ |
| `ADD COLUMN` | Yes, with limits: no `PRIMARY KEY` or `UNIQUE`; the default must be a constant; `NOT NULL` requires a non-null default; a `REFERENCES` column needs a `NULL` default when foreign keys are on |
| `DROP COLUMN` | 3.35.0+, with limits: not a primary key, unique, indexed, or foreign-key column, and not used in a view, trigger, generated column, or check |
| `ALTER COLUMN ... SET NOT NULL` / `DROP NOT NULL`; `ADD CONSTRAINT <name> CHECK (...)`, `DROP CONSTRAINT <name>` | 3.53.0+. New constraints are checked against existing rows, and the statement fails if any row violates them |
| Anything else (change a type, change a default, add a foreign key), or any of the above on older versions | Needs a table rebuild |

Check the version the application actually uses (`sqlite3 --version` for the CLI; the language binding may bundle a different one, for example `python -c 'import sqlite3; print(sqlite3.sqlite_version)'`).

### Rebuilding a table

SQLite's documented procedure, condensed:

1. Back up the database.
2. Outside any transaction: `PRAGMA foreign_keys = OFF;`
3. `BEGIN;`
4. `CREATE TABLE new_invoices (...)` with the new definition.
5. `INSERT INTO new_invoices (<columns>) SELECT <columns> FROM invoices;`
6. `DROP TABLE invoices;`
7. `ALTER TABLE new_invoices RENAME TO invoices;`
8. Recreate the table's indexes, triggers, and any views that reference it (list them first with `SELECT type, name, sql FROM sqlite_schema WHERE tbl_name = 'invoices';`, or `sqlite_master` before 3.33.0).
9. `PRAGMA foreign_key_check;` must return no rows.
10. `COMMIT;`
11. `PRAGMA foreign_keys = ON;`

Many tools (Alembic's batch mode `op.batch_alter_table`, Django, Prisma, Drizzle) generate this rebuild for you; read what they generate.

### Backups

```bash
sqlite3 app.db ".backup 'backup.db'"
sqlite3 app.db "VACUUM INTO 'backup.db'"        # 3.27.0+
```

Don't copy a live database file with `cp`: in WAL mode, recent changes live in the `-wal` file, and a copy taken mid-write can be corrupt. Check a backup with `sqlite3 backup.db 'PRAGMA integrity_check;'`.
