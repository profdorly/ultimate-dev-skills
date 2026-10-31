---
name: database-migrations
description: Use when creating, reviewing, applying, or rolling back a database migration; adding or changing columns, indexes, constraints, or tables; backfilling data; or fixing migration drift (Postgres, MySQL, SQLite; Prisma, Django, Rails, Alembic, Drizzle, Knex, TypeORM, Flyway, Liquibase, EF Core, Laravel, and more). Confirms the target database first, never runs destructive commands without approval, and uses lock-safe, backward-compatible changes.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Database Migrations

A migration runs once, against data that can't be regenerated, often while the old version of the application is still serving traffic. Mistakes show up as locked tables, broken deploys, or lost rows. Know exactly which database you're touching, make every change compatible with the code running beside it, and keep a way back.

## Rules (non-negotiable)

1. **Know the target before every command.** Before running anything that connects to a database, find out which one it will hit: the host, the database name, and the environment (see "Orient first"). If it isn't clearly a local or disposable development database, stop and get explicit approval for that exact target.
2. **Never run destructive commands without approval,** even locally, because a local database may hold work the user cares about. That covers resets, drops, `--force` flags, `db push`-style schema syncs, and anything in the danger list in [references/tools.md](references/tools.md#commands-that-destroy-data). Never set a tool's safety override on your own initiative (`PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION`, `DISABLE_DATABASE_ENVIRONMENT_CHECK=1`, Flyway `cleanDisabled=false`, Laravel `--force` in production). Use one only after the user has explicitly approved that exact operation on that exact database; Prisma's variable takes the user's consent text.
3. **Back up before any destructive or hard-to-reverse change** to a database that holds data (see "Backups"), and confirm the backup is readable before you continue.
4. **Never edit, rename, reorder, or delete a migration that has been applied anywhere** other than your own local database: shared development, CI, staging, production, or a teammate's machine via a merged branch. Write a new migration instead. If you can't tell whether it was applied, assume it was.
5. **Review every generated migration before applying it.** Generators turn renames into drop-and-add (losing the data), miss indexes and constraints, pick unsafe locking operations, and reorder things. Read the SQL.
6. **Keep the schema compatible with the code that runs alongside it.** During a deploy, old code runs against the new schema, and new code may run before the migration finishes. Removals and renames go through expand and contract (see below); a migration and the code that depends on it are separate steps.
7. **Don't lock large tables.** On a table with real traffic, use the lock-safe patterns for your database and set a lock timeout ([references/postgres.md](references/postgres.md), [references/mysql-sqlite.md](references/mysql-sqlite.md)). When you don't know the table's size or traffic, ask.
8. **Separate schema changes from data changes.** Backfills and data transformations go in their own migration or script, run in batches, are idempotent, and don't import application models that will change later.
9. **Every migration has a tested way back,** a working down migration or a written note that it's irreversible and why, plus what restoring would take. Test the down migration locally.

## Orient first (read-only)

- **Tool and conventions:** find the migration tool and directory (`migrations/`, `db/migrate/`, `prisma/migrations/`, `alembic/versions/`, `drizzle/`), how migrations are named and ordered, and whether the project commits a schema dump (`schema.rb`, `structure.sql`, `schema.prisma`, a snapshot).
- **Target database:** read the connection settings the tool will use: `DATABASE_URL` and similar variables, `.env*` files, `config/database.yml`, Django `DATABASES`, `alembic.ini` and `env.py`, `prisma.config.ts` or the datasource block, `drizzle.config.*`, `knexfile.*`, `flyway.conf`. Note which environment variable selects the environment (`RAILS_ENV`, `NODE_ENV`, `DJANGO_SETTINGS_MODULE`, `APP_ENV`, `ASPNETCORE_ENVIRONMENT`). Report the host, port, and database name, never the password:

  ```bash
  printf '%s\n' "$DATABASE_URL" | sed -E 's#://([^:/@]+):[^@]*@#://\1:***@#'
  ```

- **Engine and version:** Postgres, MySQL, MariaDB, or SQLite, and which version. Several safe patterns depend on the version.
- **Current state:** ask the tool which migrations are applied and pending (status commands in [references/tools.md](references/tools.md)). Pending migrations you didn't write, multiple heads, or drift between the schema and the migration history are reasons to stop and ask.
- **Table size and traffic:** for tables you'll alter, find out (or ask) roughly how many rows they hold and whether they take writes during deploys.

## Writing a migration

1. **Generate it with the project's tool,** using its "create only" mode when it has one so nothing is applied yet (for example `prisma migrate dev --create-only`, `drizzle-kit generate`, `alembic revision --autogenerate`, `python manage.py makemigrations`, `bin/rails generate migration`).
2. **Read the generated SQL** (or print it: `python manage.py sqlmigrate <app> <migration>`, `alembic upgrade <current>:head --sql`, `php artisan migrate --pretend`, `liquibase update-sql`, `dotnet ef migrations script`). Check for:
   - drops you didn't intend, especially a drop and add where you meant a rename;
   - operations that rewrite or lock a large table;
   - missing indexes on new foreign keys and new query filters;
   - `NOT NULL` columns without a default on tables that have rows;
   - defaults, constraint names, and types matching the project's conventions.
3. **Edit it into a safe shape** using the patterns below and the references for your engine. Hand-written SQL is fine and often necessary.
4. **Write the down migration,** or mark the migration irreversible with the reason.
5. **Apply it locally, then roll it back, then apply it again.** Confirm the schema matches what the code expects and that the test suite passes.
6. **Check the migration in alongside its schema dump** if the project commits one, so they stay in sync.

## Expand and contract

Any change that would break the running code if applied in one step becomes a sequence of separately deployed steps:

| Change | Steps |
|---|---|
| Rename a column | 1. Add the new column. 2. Write to both columns. 3. Backfill the new column in batches. 4. Read from the new column. 5. Stop writing the old one. 6. Drop the old column in a later release. |
| Change a column's type | Same as a rename: a new column with the new type, dual writes, backfill, switch reads, drop the old column. |
| Drop a column | 1. Stop using it in code (Rails: add it to `ignored_columns`; other ORMs: remove it from the model). 2. Deploy. 3. Drop it in a later migration. |
| Add a `NOT NULL` column | Add it as nullable (or with a constant default), backfill, then add the constraint with the lock-safe pattern for your engine. |
| Split or merge tables | Create the new structure, dual write, backfill, switch reads, then remove the old structure. |
| Add a unique or foreign-key constraint to existing data | Find and fix violating rows first; then add the constraint with the lock-safe pattern. |

Propose the step plan to the user. Each step is usually its own pull request or deploy.

## Data migrations and backfills

- Keep them out of schema migrations, or in their own migration file, so a slow backfill doesn't hold schema locks.
- Process rows in batches (thousands, not millions, per transaction), keyed on the primary key, and commit each batch.
- Make them idempotent: rerunning one after a partial failure must not double-apply anything (`WHERE new_column IS NULL`, upserts).
- Don't import application models into migrations: their future versions won't match this point in history. Use the migration framework's historical models (Django `apps.get_model`), a minimal model class defined inside the migration (Rails), or plain SQL.
- Estimate the run time on a realistic copy of the data before running it anywhere that matters, and report the estimate.

## Backups

Before a destructive change to a database with data, and only with the user's approval for anything not local:

```bash
pg_dump --format=custom --file="backup-$(date -u +%Y%m%dT%H%M%SZ).dump" "$DATABASE_URL"   # Postgres
mysqldump --single-transaction --routines --triggers --databases <db> > "backup-$(date -u +%Y%m%dT%H%M%SZ).sql"   # MySQL
sqlite3 app.db ".backup 'backup.db'"                                                       # SQLite
```

Check the backup: `pg_restore --list <file> | head`, `head -n 30 <file.sql>`, `sqlite3 backup.db 'PRAGMA integrity_check;'`. Keep backups out of the repository (they contain data), and tell the user where they are. Managed databases usually have snapshots or point-in-time recovery; asking the user to take one is often better than a dump.

## Resolving conflicts and drift

- **Two branches added migrations with the same number or parent:** don't renumber migrations that were applied elsewhere. Use the tool's merge mechanism (Django `makemigrations --merge`, Alembic `alembic merge heads`), or for timestamped tools, rebase your unapplied migration onto the latest. Ask before changing anything that may have been applied.
- **Drift** (the database doesn't match the migration history): find out why first; someone may have changed it by hand, or a migration was edited. Don't "fix" drift by resetting the database or marking migrations as applied (`prisma migrate resolve`, `flyway repair`, `migrate force`, `alembic stamp`) without the user's approval; those rewrite history and can hide real differences.
- **A migration failed halfway:** on MySQL, DDL isn't transactional, so part of it may have been applied. Inspect the actual schema before retrying or rolling back, and report what state it's in.

## Stop and ask when

- The target isn't clearly a local, disposable database.
- A command would drop, reset, truncate, or overwrite data, or needs a `--force` or safety override.
- A generated migration drops anything, or contains a drop-and-add that might be a rename.
- The change touches a large or busy table and you don't know its size or traffic.
- A migration that might have been applied elsewhere needs changing.
- There's drift, multiple heads, a dirty migration state, or pending migrations you didn't write.
- A change can't be made backward-compatible in one step and needs a multi-deploy plan.

## Final report

```
Migration: 0043_add_invoice_due_date (new; not applied outside my local database)
Target:    local Postgres 17 at localhost:5432/app_dev (from .env.development)
Change:    adds invoices.due_date (nullable, no default); index created concurrently
Safety:    metadata-only column add; no table rewrite; lock_timeout 5s
Reverse:   down migration drops the index and the column; tested up, down, up
Checks:    makemigrations --check clean; pytest (412 passed)
Next step: backfill due_date in batches (0044), then SET NOT NULL via a validated constraint (0045)
```

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "It's just my dev database, I'll reset it." | It may hold data the user needs, and the command might be pointed elsewhere. Check the target and ask. |
| "I'll fix the old migration instead of adding a new one." | Databases where it already ran never see your edit, and the histories diverge. |
| "The ORM generated it, so it's right." | Generators turn renames into data loss and pick locking operations. Read it. |
| "`db push` is faster than writing a migration." | It changes the schema with no history and can drop columns to match. Use migrations for anything shared. |
| "Rename the column in one migration." | The running code still uses the old name and fails until the deploy finishes. Expand and contract. |
| "Add the index normally, the table's small." | Find out. On a large table it blocks writes for the whole build. |
| "Backfill everything in one UPDATE." | One huge transaction holds locks, bloats the log, and can't resume after a failure. Batch it. |
| "Mark it as applied and move on." | That hides a real difference between the database and the history. Find out why first. |

## References

| Situation | Read |
|---|---|
| Postgres: lock levels, adding columns, `NOT NULL`, indexes, foreign keys, type changes, enums, backfills, checking for long transactions | [references/postgres.md](references/postgres.md) |
| MySQL and MariaDB online DDL; SQLite's limited `ALTER TABLE` and table rebuilds | [references/mysql-sqlite.md](references/mysql-sqlite.md) |
| Status, dry-run, apply, and rollback commands for each migration tool, and the commands that destroy data | [references/tools.md](references/tools.md) |
