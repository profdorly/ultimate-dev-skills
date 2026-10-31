# Migration tool commands

Run every command through the project's wrapper (`npx`, `pnpm exec`, `uv run`, `bundle exec`, `bin/rails`, `mix`) and only after you've confirmed which database it will connect to (SKILL.md, "Orient first"). Flags change between major versions; check `<command> --help` when in doubt. Commands marked **destroys data** are in the list at the end and need the user's explicit approval for the exact target.

## Prisma

- Status: `npx prisma migrate status` (exits 1 when migrations are pending, failed, or diverged, or the database can't be reached).
- Create without applying: `npx prisma migrate dev --create-only --name <name>`. Then edit the generated `migration.sql`.
- Apply in shared environments: `npx prisma migrate deploy`. `migrate dev` is for local development only.
- Compare schemas or print SQL: `npx prisma migrate diff --from-... --to-... --script`. The source flags differ between Prisma 6 and 7 (Prisma 7 uses `--from-config-datasource`, `--to-schema`, `--from-migrations`); check `npx prisma migrate diff -h`.
- No down migrations: roll back by writing a new migration (`migrate diff` can generate the reverse SQL).
- Watch for:
  - A rename in `schema.prisma` generates `DROP COLUMN` plus `ADD COLUMN`. Edit the SQL to `ALTER TABLE ... RENAME COLUMN` (and use expand and contract if code is running).
  - `migrate dev` resets the database when it detects drift or a history conflict. If it says it needs to reset, stop and ask.
  - Prisma blocks some destructive commands when it detects an AI agent, and asks for `PRISMA_USER_CONSENT_FOR_DANGEROUS_AI_ACTION` to be set to the user's consent message. Stop, explain the operation, and set it only after the user explicitly consents to that exact command and database.
- `migrate resolve --applied` and `--rolled-back` rewrite the migration history table. Approval required.

## Django

- Status: `python manage.py showmigrations [app]`; plan: `python manage.py migrate --plan`.
- Check that models and migrations agree: `python manage.py makemigrations --check --dry-run`.
- Create: `python manage.py makemigrations <app> --name <name>`.
  - If it asks "Did you rename ...?", stop and ask the user. With `--noinput`, Django assumes no rename and generates a remove plus an add, which loses the column's data.
- Show SQL: `python manage.py sqlmigrate <app> <migration>`.
- Apply: `python manage.py migrate [<app> [<migration>]]`.
- Roll back: `python manage.py migrate <app> <previous_migration>`.
- Merge parallel migrations: `python manage.py makemigrations --merge`.
- Data migrations: `RunPython` with a reverse function, using `apps.get_model(...)`, never direct model imports.
- Concurrent indexes on Postgres: `atomic = False` on the migration and `AddIndexConcurrently` / `RemoveIndexConcurrently`.

## Rails (Active Record)

- Status: `bin/rails db:migrate:status`.
- Create: `bin/rails generate migration AddDueDateToInvoices due_date:date`.
- Apply: `bin/rails db:migrate`.
- Roll back: `bin/rails db:rollback STEP=1`, or `bin/rails db:migrate:down VERSION=<timestamp>`.
- Concurrent indexes on Postgres: `disable_ddl_transaction!` and `add_index ..., algorithm: :concurrently`.
- Use the `strong_migrations` gem's checks if the project has it.
- Data in migrations: define a minimal model class inside the migration rather than using the application's models.

## Alembic (SQLAlchemy)

- Status: `alembic current`, `alembic history --verbose`, `alembic heads` (more than one head means parallel branches).
- Check models against the database: `alembic check` (Alembic 1.9+).
- Create: `alembic revision --autogenerate -m "<message>"`. Autogenerate doesn't detect renames (they appear as drop and add), misses some server-default and type changes, and can't name anonymous constraints. Read and edit the result.
- Show SQL without connecting: `alembic upgrade <current_rev>:head --sql`.
- Apply: `alembic upgrade head`. Roll back one: `alembic downgrade -1`.
- Merge parallel heads: `alembic merge heads -m "<message>"`.
- `alembic stamp` rewrites the version table without running migrations. Approval required.
- SQLite: use `with op.batch_alter_table(...)` for changes that need a table rebuild.

## Drizzle Kit

- Create: `npx drizzle-kit generate`. It may ask interactively whether a column or table was renamed or created; that prompt can hang a non-interactive session. If it asks, stop and ask the user, or write the SQL by hand: `npx drizzle-kit generate --custom --name <name>` creates an empty migration file.
- Apply: `npx drizzle-kit migrate`.
- Check generated migrations for collisions: `npx drizzle-kit check`.
- Print the schema as SQL: `npx drizzle-kit export`.
- No down migrations: roll back with a new migration.
- Multiple configs per environment are common (`--config=drizzle-prod.config.ts`); confirm which config a command uses.

## Knex

- Status: `npx knex migrate:status` (or `migrate:list`).
- Create: `npx knex migrate:make <name>`.
- Apply: `npx knex migrate:latest`; one step: `npx knex migrate:up`.
- Roll back: `npx knex migrate:down` undoes one migration. `npx knex migrate:rollback` undoes the **whole last batch**, which can be several migrations.
- Disable the wrapping transaction for one migration: `exports.config = { transaction: false };`.

## TypeORM

- Status: `npx typeorm migration:show -d <data-source-file>`.
- Create: `npx typeorm migration:generate <path/Name> -d <data-source-file>` (review it), or `migration:create` for an empty one.
- Apply: `npx typeorm migration:run -d <data-source-file>`. Roll back one: `npx typeorm migration:revert -d <data-source-file>`.
- `synchronize: true` in the data source changes the schema automatically at startup; keep it off for anything but throwaway databases.

## Sequelize

- Status: `npx sequelize-cli db:migrate:status`.
- Create: `npx sequelize-cli migration:generate --name <name>`.
- Apply: `npx sequelize-cli db:migrate`. Roll back one: `npx sequelize-cli db:migrate:undo`.
- `sequelize.sync({ force: true })` drops tables and `sync({ alter: true })` changes them to match models; neither belongs in shared environments.

## Flyway

- Status: `flyway info`; verify checksums and order: `flyway validate`.
- Apply: `flyway migrate`.
- Undo migrations (`flyway undo`) exist only in some editions; otherwise roll back with a new versioned migration.
- `flyway repair` rewrites the schema history table, and `flyway baseline` marks a starting version. Approval required.

## Liquibase

- Status: `liquibase status --verbose`.
- Show SQL: `liquibase update-sql`; for a rollback, `liquibase rollback-count-sql --count=1`.
- Apply: `liquibase update`. Roll back: `liquibase rollback-count --count=1`.
- `changelog-sync` marks every changeset as applied without running it, and `clear-checksums` resets validation. Approval required.

## golang-migrate

- Status: `migrate -path <dir> -database "$DATABASE_URL" version`.
- Create: `migrate create -ext sql -dir <dir> -seq <name>` (creates up and down files).
- Apply: `migrate -path <dir> -database "$DATABASE_URL" up` (or `up 1`).
- Roll back one: `migrate -path <dir> -database "$DATABASE_URL" down 1`. Always pass a number: `down` without one rolls back everything.
- A failed migration leaves the version "dirty". `migrate force <version>` clears it without running anything; inspect the schema first and get approval.

## goose

- Status: `goose -dir <dir> <driver> "$DSN" status` (or set `GOOSE_DRIVER` and `GOOSE_DBSTRING`).
- Create: `goose -dir <dir> create <name> sql`.
- Apply: `goose ... up`, or `up-by-one`. Roll back one: `goose ... down`.
- Validate files: `goose -dir <dir> validate`.

## sqlx (Rust)

- Status: `sqlx migrate info`.
- Create a reversible pair: `sqlx migrate add -r <name>`.
- Apply: `sqlx migrate run`. Roll back one: `sqlx migrate revert`.
- Offline query metadata: after schema changes, `cargo sqlx prepare` if the project uses offline mode.

## Diesel (Rust)

- Status: `diesel migration list`. Create: `diesel migration generate <name>`.
- Apply: `diesel migration run`. Roll back one: `diesel migration revert`. Check the down migration: `diesel migration redo`.

## Ecto (Elixir)

- Status: `mix ecto.migrations`. Create: `mix ecto.gen.migration <name>`.
- Apply: `mix ecto.migrate`. Roll back one: `mix ecto.rollback --step 1`.
- Concurrent indexes on Postgres: `@disable_ddl_transaction true` and `@disable_migration_lock true` in the migration, with `create index(..., concurrently: true)`.

## Entity Framework Core

- Status: `dotnet ef migrations list`.
- Create: `dotnet ef migrations add <Name>`; review the generated `Up` and `Down` (renames sometimes come out as drop and add).
- Show SQL: `dotnet ef migrations script <FromMigration> <ToMigration> --idempotent -o migrate.sql`.
- Apply: `dotnet ef database update`. Roll back: `dotnet ef database update <PreviousMigration>`.
- `dotnet ef migrations remove` deletes the last migration file; only use it for a migration that was never applied anywhere.

## Laravel

- Status: `php artisan migrate:status`. Create: `php artisan make:migration <name>`.
- Show SQL without running: `php artisan migrate --pretend`.
- Apply: `php artisan migrate`. Roll back: `php artisan migrate:rollback --step=1`.
- In production, Laravel asks for confirmation; `--force` skips it. Don't add `--force` yourself.

## Commands that destroy data

Never run these without the user's explicit approval for the exact database, and a verified backup when the data matters:

| Tool | Commands |
|---|---|
| Prisma | `prisma migrate reset`, `prisma db push --accept-data-loss` (and `db push` on any shared database), `prisma migrate dev` when it offers to reset |
| Django | `manage.py flush`, `manage.py migrate <app> zero`, `manage.py reset_db` (django-extensions) |
| Rails | `db:drop`, `db:reset`, `db:purge`, `db:truncate_all`, `db:schema:load` and `db:setup` on an existing database, `db:seed:replant`; setting `DISABLE_DATABASE_ENVIRONMENT_CHECK=1` |
| Alembic | `alembic downgrade base` |
| Drizzle | `drizzle-kit push` on any database with data |
| Knex | `knex migrate:rollback --all` |
| TypeORM | `typeorm schema:drop`, `typeorm schema:sync`, `synchronize: true` |
| Sequelize | `db:migrate:undo:all`, `db:drop`, `sync({ force: true })` |
| Flyway | `flyway clean` (keep `cleanDisabled=true`, the default) |
| Liquibase | `drop-all` |
| golang-migrate | `down` without a count, `drop` |
| goose | `goose reset`, `goose down-to 0` |
| sqlx | `sqlx database drop`, `sqlx database reset` |
| Diesel | `diesel database reset` |
| Ecto | `mix ecto.drop`, `mix ecto.reset` |
| EF Core | `dotnet ef database drop` |
| Laravel | `migrate:fresh`, `migrate:refresh`, `migrate:reset`, `db:wipe` |
| Any SQL | `DROP DATABASE`, `DROP SCHEMA ... CASCADE`, `DROP TABLE`, `TRUNCATE`, `DELETE` or `UPDATE` without a `WHERE` |
