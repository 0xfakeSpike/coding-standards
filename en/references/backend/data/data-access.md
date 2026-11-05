# Data access

This document covers how to organize data access code: where table definitions and usage live, and how business code uses them. For general layering rules, see [layering.md](../architecture/layering.md). For query optimization, see [indexes-and-queries.md](indexes-and-queries.md).

## Single entry point

When a column or table has no readers, handle the model, repo, and migration together. See [schema-design.md](schema-design.md).

A table belongs to exactly one module. For the exception for cross-module read-only queries, see [service-boundaries.md](../architecture/service-boundaries.md).

### Give each table one model/repo entry point [Required]

Keep each table's definition and all its reads and writes in one place. The model describes the table structure, and the repo provides every operation on the table. Go projects put this in the business package's `store.go`. See [project.md](../../languages/go/project.md). Other languages may use different directory names, but the structure is the same.

**Why:** When a table's reads and writes are scattered, changing one column takes a global search, and it is unclear how the table is actually used.

**Example:**

Avoid:

```text
handlers/order_handler     inline SELECT and UPDATE statements
jobs/close_orders          another inline UPDATE statement
pages/order_list           inline JOIN statement
```

Prefer:

```text
order/order               column definitions for orders and order_items
order/store               all queries and writes for orders and order_items
handlers/order_handler     calls the business layer
jobs/close_orders          calls the business layer
```

**Source:** Team

### Keep table definitions and usage traceable [Required]

Starting from a table name, you can see the column definitions in one place and every usage in the repo's method list. From each method's callers, you can then see who uses it.

To make this work:

- SQL and ORM calls appear only in the repo.
- Do not build table names by dynamic string concatenation, or they cannot be searched.
- Sharded-table routing logic also lives in the repo.

**Tooling:** In CI, search for SQL keywords and ORM calls outside the repo directory, and reject the merge if any are found.

**Source:** Team

### Name repo methods by purpose [Recommended]

Method names state what is queried and by what condition. Do not expose a generic condition-building entry point to the business layer.

| Avoid | Prefer |
|---|---|
| `query(table, conditions)` | `find_unpaid_by_user(user_id)` |
| `update(id, fields)` | `mark_paid(order_id, paid_at)` |

**Why:** A generic entry point makes every call look the same. That is no better than having no entry point, and the method list no longer shows how the table is used.

## Responsibility boundaries

API responses use separate structures. See [layering.md](../architecture/layering.md).

### Keep business decisions out of the repo [Recommended]

The repo only stores and retrieves data. It does not evaluate business state or call external services. Business rules live in the business layer.

**Example:**

Avoid: `mark_paid` checks internally whether the order has expired and sends a notification.

Prefer: the business layer decides whether the order can be paid, and the repo only runs a conditional update and returns the affected row count.

### Apply cross-cutting filters in the repo [Required]

The repo or its infrastructure adds tenant filters and soft-delete filters in one place. The business layer does not repeat them. Methods that bypass a filter must say so in their name, for example `find_including_deleted`.

**Why:** Missing one filter leaks data or reads deleted data.

### Let the business layer set transaction boundaries [Recommended]

The repo provides methods that can join a transaction, and does not decide cross-operation transactions on its own. The business layer opens a transaction through a transaction handle or unit of work and passes it to the repo. This does not count as taking ORM calls out of the repo. For how to write transactions, see [transactions.md](transactions.md).

**Why:** One business operation may span several repos, and only the business layer knows which writes must succeed together.

## ORM boundaries

### Keep ORM query objects inside the repo [Recommended]

Do not pass ORM query builders or lazy-loaded objects outside the repo. The repo returns fully loaded data. The business layer may hold a transaction handle or unit of work.

**Why:** Once a lazy-loaded object leaves the repo, a single field access can trigger a query, causing N+1 or an error after the connection closes.

### Do not force the ORM onto every scenario [Recommended]

Use the ORM for simple reads and writes. Write SQL directly in the repo for complex queries and batch operations. See [indexes-and-queries.md](indexes-and-queries.md). Both share the same entry point, and callers do not need to know which one is underneath.

### Treat migration files as the schema authority [Required]

**Applies to:** production service

The migration tool manages the production schema. Do not let the ORM create or alter tables at startup. Model definitions must match the migration result.

**Tooling:** After the empty-database check in CI, compare the model definitions with the actual table structure. See [migrations.md](migrations.md).

## Read replicas

### Choose read replicas in the repo [Recommended]

With read/write splitting, the repo's methods decide whether to read the primary or a replica. The business layer does not pick connections directly. Paths that must see the result right after a write read the primary.

**Why:** Replicas have replication lag, so reading a replica right after a write may miss the data just written.

## Testing

### Test the repo with a real database [Recommended]

SQL dialects and transaction behavior vary by database. Repo tests use the same database type as production. See [testing.md](../testing.md).
