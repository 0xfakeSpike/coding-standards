# Schema design

Focuses on relational databases (MySQL, PostgreSQL). For specific syntax, follow the database in use. For indexes, see [indexes-and-queries.md](indexes-and-queries.md). For schema changes, see [migrations.md](migrations.md).

## Choosing tables and columns

### Keep tables few [Required]

Put data of the same kind in one table and use columns to distinguish type or state. Do not create a separate table for each type or state.

**Why:** More tables mean more queries, migrations, and data access code, and the same rule tends to get written separately on several tables.

**Example:**

Avoid: three tables with the same structure, `system_notices`, `promo_notices`, and `archived_notices`.

Prefer: one `notices` table, with `type` for the type and `status` for whether it is archived.

**Exceptions:** Do not merge data of different concepts. See "Align tables with concept layers".

**Source:** Team

### Keep columns minimal [Required]

Question whether each column is necessary: who writes it, who reads it, and what happens if it is not stored. If you cannot answer, do not add it. Keep base columns such as `created_at` and `updated_at`.

**Why:** Extra columns must be continually written, validated, and migrated, and they mislead people into thinking they carry business meaning.

**Source:** Team

### Align tables with concept layers [Required]

Table division and hierarchy follow the layers of business concepts. One concept maps to one table, and parent-child relationships are expressed with foreign key columns. Do not create intermediate tables that map to no concept, and do not squeeze concepts from different layers into one table.

**Example:**

Avoid: putting `item1_sku` and `item2_sku` in the `orders` table, which squeezes orders and order lines into one layer.

Prefer: two tables, `orders` and `order_items`, with `order_items.order_id` pointing to the parent.

For how to divide concepts, see [domain-modeling.md](../architecture/domain-modeling.md).

**Source:** Team

### Keep business logic out of the database [Required]

Business rules go in the application's business layer, not in triggers, stored procedures, stored functions, computed logic in views, or database scheduled events. The database is responsible only for storage and data integrity. For layering, see [layering.md](../architecture/layering.md).

| Belongs in the database | Does not belong in the database |
|---|---|
| `NOT NULL`, unique constraints, foreign keys, value-range `CHECK` | State transitions, amount calculations, permission checks |
| Indexes, column defaults | Using triggers to sync other tables, send notifications, or write business logs |
| Temporary objects a migration tool creates while running | Using stored procedures to orchestrate business flows |

**Why:** Logic in the database is not reviewed and tested with application code, callers cannot see its side effects, and it is hard to trace during troubleshooting.

**Exceptions:** Constraints are not business logic and should be used. Business uniqueness must be guaranteed by a unique index.

**Source:** Team, Azure: Busy Database

### Delete columns and tables with no readers [Required]

Once you confirm a column or table has no readers, delete it outright. Do not keep writing to it or keep it around. For how to decide and how to confirm for production data, see [deletion.md](../../general/deletion.md).

**Source:** Team

### Write comments on tables and columns [Recommended]

**Applies to:** team project

Write comments on tables and columns, and list the meaning of each value for enum columns. This is metadata stored in the database, not code comments, so it is not subject to the code comment rules.

**Example:**

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY COMMENT 'Order ID',
  status VARCHAR(20) NOT NULL COMMENT 'Status: pending = awaiting payment, paid = payment received, closed = closed'
) COMMENT = 'Orders';
```

PostgreSQL uses a separate statement:

```sql
COMMENT ON COLUMN orders.status IS 'Status: pending = awaiting payment, paid = payment received, closed = closed';
```

## Naming

### Use uniform naming rules [Recommended]

Table and column names use lowercase `snake_case` and avoid reserved words. Use singular or plural table names consistently across the whole database.

| Object | Rule | Example |
|---|---|---|
| Primary key | `id` | `id` |
| Foreign key column | `<singular_table_name>_id` | `user_id` |
| Boolean column | Starts with `is_` | `is_active` |
| Time column | Ends with `_at` | `paid_at` |
| Regular index | `idx_table_column` | `idx_orders_user_id` |
| Unique index | `uk_table_column` | `uk_users_email` |

## Base columns and types

### Give every table a primary key and time columns [Required]

Every table has a primary key plus `created_at` and `updated_at`. These are the basis for troubleshooting and data sync.

### Do not use floating-point types for money [Required]

Store amounts as `DECIMAL` or as `BIGINT` in cents. Do not use `FLOAT` or `DOUBLE`.

**Why:** Floating-point numbers cannot represent decimal fractions exactly, and errors appear as values accumulate.

### Store times in UTC [Required]

Store columns that represent a moment in time in UTC. In PostgreSQL, use `timestamptz`, which stores UTC internally and does not keep the original time zone. MySQL's `TIMESTAMP` tops out in 2038 and converts with the connection time zone. For long-lived columns, use `DATETIME` and agree to write UTC.

**Exceptions:** Local wall-clock times such as business hours do not represent a moment in time. Use a type without a time zone.

**Source:** MySQL Manual, PostgreSQL Docs

### Use utf8mb4 as the MySQL character set [Required]

Do not use `utf8mb3` (formerly `utf8`). It uses at most 3 bytes per character, supports only the Basic Multilingual Plane, and cannot store supplementary characters such as emoji.

**Source:** MySQL Manual

### Make columns non-null by default [Recommended]

Columns default to `NOT NULL` with a suitable default value. Use `NULL` only for values that are truly "unknown", because it complicates comparisons and aggregation.

### Choose the smallest suitable type [Recommended]

Choose types by value range. Do not define every column as `VARCHAR(255)`. Columns used in comparisons and joins keep the same type and collation.

### Do not use JSON columns in place of modeling [Recommended]

Data that needs querying, constraints, or joins goes in its own columns. JSON stores only content that is read and written as a whole and whose structure truly is not fixed.

## Normalization and denormalization

### Follow third normal form by default [Recommended]

Store each fact in one place. Denormalization requires a measured read bottleneck as its basis, and you must state who maintains the redundant column.

**Why:** Redundant data needs syncing, and when a sync is missed the two copies contradict each other.

**Example:** Business decisions rely on the source-of-truth data, not redundant counts or states. See [consistency.md](../reliability/consistency.md).

## Primary and foreign keys

### Keep primary keys short and roughly increasing [Recommended]

Use `BIGINT` auto-increment, snowflake IDs, or UUIDv7 for primary keys. Random UUIDv4 has poor write performance as a clustered primary key. When exposing IDs externally, avoid enumerable auto-increment IDs.

**Why:** Every InnoDB secondary index record includes the primary key, so a longer primary key makes all secondary indexes larger.

**Source:** MySQL Manual

### Decide foreign key constraints once per project [Recommended]

Foreign keys guarantee integrity in small systems. High-concurrency or sharded systems usually move this to the application layer. Record the choice in project docs, and do not mix approaches within one project.

**Example:** `gh-ost` does not support tables with foreign keys. Consider this early if you plan to use it for large-table changes.

**Source:** gh-ost Docs

## Soft delete

### Use soft delete only when needed [Recommended]

Do not add soft delete to every table by default. When using `deleted_at`, note:

- All queries must filter on it. Handle this in the data access layer. See [data-access.md](data-access.md).
- Unique constraints must take soft delete into account.
- Data that compliance requires to be fully deleted cannot only be soft-deleted. See [privacy.md](../security/privacy.md).

In PostgreSQL, use a partial unique index that constrains only rows not deleted:

```sql
CREATE UNIQUE INDEX uk_users_email ON users (email) WHERE deleted_at IS NULL;
```

MySQL unique indexes allow multiple `NULL` values, so adding a nullable `deleted_at` does not constrain rows not deleted. Use a non-null deletion marker column instead: 0 when not deleted, and the primary key value when deleted:

```sql
ALTER TABLE users
  ADD COLUMN deleted_mark BIGINT NOT NULL DEFAULT 0,
  ADD UNIQUE KEY uk_users_email (email, deleted_mark);
```

**Source:** MySQL Manual, PostgreSQL Docs

## Multi-tenancy

### Include tenant_id on tenant data in shared databases [Required]

In a shared database, every tenant data table has `tenant_id`, placed leftmost in unique indexes and common indexes. The data access layer adds the filter in one place. Do not rely on each caller remembering it.

**Why:** Missing one filter is a cross-tenant data leak.

### Do not create tables or add columns per tenant [Recommended]

Do not create separate tables for each tenant, and do not add columns for a single tenant's needs.

**Why:** As tenants grow, the number of tables and custom columns becomes unmanageable.

**Source:** Azure: Multitenant Storage

### Choose a scheme by isolation requirements [Recommended]

Default to shared tables. Use separate databases or sharding only when compliance requires physical isolation or a particular tenant's load affects others. PostgreSQL row-level security can be a second line of defense, but table owners bypass policies by default and need `FORCE ROW LEVEL SECURITY`, and superusers and `BYPASSRLS` roles always bypass them.

**Source:** Azure: Multitenant Storage, PostgreSQL Docs

## Sharding

### Do not shard if a single database is enough [Recommended]

First try index optimization, archiving historical data, and read replicas. Sharding brings the long-term costs of cross-shard queries, cross-shard transactions, and data rebalancing.

### Choose the shard key carefully [Recommended]

The shard key must be immutable, high-cardinality, and evenly distributed, and it must match the main queries so that most requests land on a single shard. Avoid auto-increment values, sequential times, low-cardinality columns, and frequently changing columns.

**Why:** Changing the shard key after shipping usually means migrating all the data. A wrong choice also creates hot shards.

**Example:** Map many logical shards onto fewer physical nodes, so rebalancing only needs a mapping change.

**Source:** Azure: Sharding Pattern
