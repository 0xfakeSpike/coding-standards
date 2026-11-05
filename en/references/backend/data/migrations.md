# Database migrations

This document covers migration tools, migration file management, empty-database verification, large-table change risks, and rollback. For the release process, see [release.md](../../general/release.md).

## Migration tools

### Manage schema changes with a migration tool [Required]

**Applies to:** production service

Write every schema change as a migration file, keep it under version control, and let the tool record execution history. Common tools are Flyway, Liquibase, Alembic, Prisma Migrate, and golang-migrate.

**Why:** Manual changes cannot be tracked, and schemas gradually drift apart between environments.

**Source:** Flyway Docs, Azure: Multitenant Storage

### Do not change production schemas by hand [Required]

**Applies to:** production service

Run production schema changes only through the migration tool. Changes made during a production emergency must be added as migration files afterwards, and all environments brought in line.

### Do not modify applied migration files [Required]

When something needs changing, write a new migration. The tool uses checksums to detect changes to executed files.

**Why:** After an executed file is modified, new and old environments no longer have the same schema.

**Tooling:** Flyway's `validate` compares the checksums of local files against the execution records.

**Source:** Flyway Docs

## Migration files

### Make file names show order and content [Recommended]

File names contain a version and a description. The version is unique, and the description states what this change does.

**Example:**

Avoid:

```text
update.sql
fix2.sql
```

Prefer:

```text
V20261105093000__add_status_to_orders.sql
V20261105101500__create_order_items.sql
```

When several people develop in parallel, use timestamps as versions to reduce number collisions.

**Source:** Flyway Docs

### Do one thing per migration [Recommended]

Each file contains one logical change. Keep schema changes and data changes separate.

**Why:** A migration can fail at any statement. The smaller the file, the easier it is to tell where it stopped after a failure.

**Source:** Flyway Docs

### Know the database's DDL transaction behavior [Recommended]

PostgreSQL DDL can roll back inside a transaction, so a failed migration is undone. MySQL commits implicitly before and after DDL, so statements already executed are not undone on failure. The tool can only mark the failure, and you must clean up by hand and repair the history.

Statements that cannot run in a transaction go in their own file, configured to run outside a transaction, such as PostgreSQL's `CREATE INDEX CONCURRENTLY`.

**Tooling:** Flyway uses the script configuration `executeInTransaction=false` to control whether a script runs in a transaction, and `repair` to clean up failed records.

**Source:** Flyway Docs, PostgreSQL Docs

## Empty-database verification

### Run all migrations from an empty database in CI [Recommended]

**Applies to:** team project

Whenever migration files change, CI runs them on an empty database from the start to the latest version, then runs the tests that depend on the database. Use the same database type as production, not an in-memory database.

**Why:** Only a migration set that can rebuild from an empty database can be used for new environments, test environments, and disaster recovery.

**Example:**

```bash
flyway clean
flyway migrate
flyway validate
```

### Do not use clean commands in production [Required]

Commands like `flyway clean` delete all objects. Use them only in development and test environments, and disable them in production configuration.

**Source:** Flyway Docs

## Large-table changes

### Assess locks and rewrites before changing large tables [Required]

**Applies to:** production service

Before shipping, confirm whether the statement blocks reads and writes and whether it rewrites the whole table. Rehearse on data close to production scale, and run it during off-peak hours.

| Operation | MySQL InnoDB | PostgreSQL |
|---|---|---|
| Add a secondary index | `INPLACE`, allows concurrent reads and writes | Plain form blocks writes; use `CONCURRENTLY` |
| Add a column | `INSTANT` by default, changes metadata only; requires MySQL 8.0.12 or later | No rewrite with a non-volatile default, requires PostgreSQL 11 or later; a volatile default rewrites the whole table |
| Drop a column | `INSTANT` by default; requires MySQL 8.0.29 or later | Only marks it as dropped, no table rewrite, but needs a brief exclusive lock |
| Change a column type | `COPY`, blocks writes and rebuilds the table | Usually rewrites the table and indexes, temporarily needing about twice the disk |
| Add a constraint | Depends on the constraint type | Add the constraint with `NOT VALID`, then `VALIDATE CONSTRAINT` |

**Source:** MySQL Manual, PostgreSQL Docs

### Declare the algorithm explicitly in MySQL [Recommended]

State `ALGORITHM` and `LOCK` in `ALTER TABLE`. If the conditions are not met, the statement fails outright instead of silently degrading to a heavier method.

**Example:**

```sql
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '', ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_orders_source (source), ALGORITHM=INPLACE, LOCK=NONE;
```

The number of `INSTANT` column additions and drops has a limit, and once it is reached the table must be rebuilt. Tables with a compressed row format or a full-text index do not support instant column addition.

**Source:** MySQL Manual

### Set a lock wait timeout for DDL [Required]

**Applies to:** production service

DDL must wait for transactions holding the table's metadata lock to finish. While it waits, ordinary queries behind it queue too, so one simple DDL statement can make the whole table unavailable. Check for long transactions before running, set a short timeout, and retry on failure.

**Example:**

MySQL:

```sql
SET SESSION lock_wait_timeout = 5;
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '', ALGORITHM=INSTANT;
```

PostgreSQL:

```sql
SET lock_timeout = '5s';
ALTER TABLE orders ADD COLUMN source VARCHAR(20) NOT NULL DEFAULT '';
```

**Tooling:** In MySQL, check `performance_schema.metadata_locks`. In PostgreSQL, check `pg_locks`.

**Source:** MySQL Manual, PostgreSQL Docs

### Clean up after a failed concurrent index build [Recommended]

`CREATE INDEX CONCURRENTLY` is slower than a normal index build. It scans the table twice and waits for existing transactions to finish, and only one can run on a table at a time. On failure it leaves an `INVALID` index: queries do not use it, but writes still maintain it. Drop it and rebuild.

**Example:**

```sql
DROP INDEX CONCURRENTLY idx_orders_source;
CREATE INDEX CONCURRENTLY idx_orders_source ON orders (source);
```

**Source:** PostgreSQL Docs

### Use online tools for changes native DDL cannot handle [Recommended]

**Applies to:** production service

For large-table changes on MySQL that block writes, use `gh-ost` or `pt-online-schema-change`. These tools create a shadow table, copy data in batches, and finally swap the table names.

Before using `gh-ost`, confirm:

- Row-format binary logs are available.
- The table has no foreign keys or triggers.
- The tables before and after the change have the same primary key or non-null unique key.
- The disk can hold a copy of the shadow table.

Verify on a replica first with `--test-on-replica`. Use throttling to control the impact on the primary, and use `--postpone-cut-over-flag-file` to postpone the cut-over until someone is on hand.

**Source:** gh-ost Docs

## Data repair

### Back up and rehearse data repair scripts first [Required]

**Applies to:** production service

Scripts that change production data must back up first and run on a replica or rehearsal environment. Before running, confirm the affected row count with `SELECT`, and run in batches. See [transactions.md](transactions.md). Before deleting production data, confirm according to [deletion.md](../../general/deletion.md).

## Rollback

### State the rollback method before releasing [Required]

**Applies to:** production service

Before release, decide for each migration how to handle failure: what the undo statements are, whether it is reversible, and how long it takes. Dropping columns, dropping tables, and changing types are irreversible and can only be recovered from backup. Confirm the backup is usable before running them.

### Do not treat undo scripts as the only rollback method [Recommended]

An undo migration can only handle a migration that ran completely. On databases without DDL transactions, a migration can fail at any statement, and an undo script cannot help. For deletion changes, an undo script cannot restore the data either.

**Source:** Flyway Docs

### Rehearse backup and restore regularly [Required]

**Applies to:** production service

Backup restore does not depend on a specific schema, and it is the last resort when any migration fails. A backup counts as usable only once it has actually been restored successfully. For large data volumes, prefer storage snapshots.

**Source:** Flyway Docs

### Fix a shipped migration with a new migration [Recommended]

When a migration that already ran successfully in production has a problem, write a new migration to correct it. Do not modify the old file or change the database by hand.
