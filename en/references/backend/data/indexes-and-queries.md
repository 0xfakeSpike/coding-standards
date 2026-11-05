# Indexes and queries

Focuses on MySQL and PostgreSQL. For index naming, see [schema-design.md](schema-design.md). For the risks of building indexes in production, see [migrations.md](migrations.md).

## Index design

Guarantee business uniqueness with unique indexes. See [consistency.md](../reliability/consistency.md).

### Design indexes from real queries [Recommended]

List the actual queries first, then design indexes and verify them with the execution plan. Do not index every column.

**Why:** Every index takes space, adds to the optimizer's selection cost, and slows down writes.

**Source:** MySQL Manual

### Put equality before range in composite indexes [Recommended]

Follow the leftmost prefix rule. Put columns with equality conditions first and columns with range conditions after them, and account for sort columns as well. Columns after the first range condition no longer narrow the scan.

**Example:**

Query:

```sql
SELECT id, status FROM orders
WHERE user_id = ? AND status = ?
ORDER BY created_at DESC
LIMIT 20;
```

Avoid:

```sql
CREATE INDEX idx_orders_created_user ON orders (created_at, user_id, status);
```

Prefer:

```sql
CREATE INDEX idx_orders_user_status_created ON orders (user_id, status, created_at);
```

**Source:** PostgreSQL Docs, Use The Index, Luke

### Limit composite index columns [Recommended]

A single-column index is often enough. Indexes with more than three columns rarely pay off unless the query pattern is very fixed. When `(a, b)` exists, `(a)` is usually redundant.

**Source:** PostgreSQL Docs

### Do not defeat indexes [Recommended]

Do not apply functions or calculations to indexed columns. Avoid implicit type conversions and leading wildcards such as `LIKE '%abc'`.

**Example:**

Avoid:

```sql
SELECT id FROM orders WHERE DATE(created_at) = '2026-11-05';
SELECT id FROM users WHERE phone = 13800000000;
```

Prefer:

```sql
SELECT id FROM orders
WHERE created_at >= '2026-11-05' AND created_at < '2026-11-06';
SELECT id FROM users WHERE phone = '13800000000';
```

### Use partial indexes for small sets of hot data [Recommended]

In PostgreSQL, when frequently accessed rows are a small minority, index only those rows. The index is used only when the query condition implies the index condition. Parameterized conditions may not match.

**Source:** PostgreSQL Docs

### Delete indexes no longer used [Required]

Delete old indexes that have no readers. See [deletion.md](../../general/deletion.md). Primary keys and unique indexes enforce constraints, so a scan count of 0 is no reason to delete them.

In PostgreSQL, check `pg_stat_user_indexes.idx_scan`, and confirm that the statistics go back far enough to cover periodic jobs. Each replica's statistics are independent. In MySQL, you can first make the index invisible, observe, then delete it:

```sql
ALTER TABLE orders ALTER INDEX idx_orders_status INVISIBLE;
```

Invisible indexes are still maintained on writes, and unique indexes still take effect.

**Source:** Team, MySQL Manual, PostgreSQL Docs

## Execution plans

### Check the execution plan before shipping [Recommended]

**Applies to:** production service

For new or changed important queries, check the execution plan on data close to production scale. Results from small tables do not carry over to large tables.

In PostgreSQL, focus on comparing estimated and actual rows in `EXPLAIN ANALYZE`. A large gap usually means outdated statistics, so run `ANALYZE` first. Actual values on loop nodes are per-loop averages. Multiply them by `loops`.

**Source:** MySQL Manual, PostgreSQL Docs

### EXPLAIN ANALYZE really executes the statement [Required]

When using it on write statements, wrap it in a transaction and roll back.

**Example:**

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE orders SET status = 'closed' WHERE id = 1001;
ROLLBACK;
```

**Source:** PostgreSQL Docs

## Writing queries

Always parameterize SQL. See [input-and-injection.md](../security/input-and-injection.md).

### Do not write SELECT * [Recommended]

Production code lists columns explicitly. This reduces data transfer and avoids surprises from column changes.

### Limit rows returned by queries [Required]

**Applies to:** production service

Use pagination or streaming for large result sets. Do not load them into memory all at once.

### Avoid N+1 queries [Recommended]

**Applies to:** production service

Use JOIN, batched `IN` queries, or the ORM's explicit eager loading.

**Why:** The main cost is the network round trip of each query. Even with the same amount of data returned, it is much slower than one query.

**Example:**

Avoid: querying the user once per order in a loop, 1 + N queries in total.

```sql
SELECT id, name FROM users WHERE id = ?;
```

Prefer: collecting the IDs and querying in a batch, 2 queries in total.

```sql
SELECT id, name FROM users WHERE id IN (?, ?, ?);
```

**Tooling:** Turn on SQL logging in the development environment, or assert the query count in tests.

**Source:** Use The Index, Luke

## Pagination

### Use cursor conditions for deep pagination [Recommended]

Use the sort values of the last row of the previous page as the boundary, and avoid large `OFFSET` values. The cost is that you cannot jump to an arbitrary page. For pagination conventions at the API layer, see [rest.md](../api/rest.md).

**Why:** `OFFSET` must count every skipped row, so later pages get slower, and new data shifts pages out of position.

**Example:**

Avoid:

```sql
SELECT id, created_at FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;
```

Prefer:

```sql
SELECT id, created_at FROM orders
WHERE created_at < ? OR (created_at = ? AND id < ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

PostgreSQL can write this as `(created_at, id) < (?, ?)` and use the index. MySQL cannot use this row-value comparison for index access, so use the expanded form above.

**Source:** Use The Index, Luke

### Make pagination sort order fully deterministic [Required]

When sort columns can have duplicates, append the primary key as the last sort column, and make the index match. Otherwise rows with equal values get lost or repeated across pages.

**Source:** Use The Index, Luke

## Slow queries

### Enable the slow query log and address it regularly [Recommended]

MySQL's `slow_query_log` is off by default, and `long_query_time` defaults to 10 seconds. Lower it to fit the project's latency target. When addressing slow queries, sort by total time and handle the statements with the largest share first, not just the slowest single execution.

**Tooling:** In MySQL, use `mysqldumpslow` to summarize the slow log. You can enable `log_queries_not_using_indexes` together with its throttling parameter. In PostgreSQL, use `log_min_duration_statement` and `pg_stat_statements`.

**Source:** MySQL Manual

## ORM usage boundaries

For how work is divided between the ORM and raw SQL, see [data-access.md](data-access.md).

### Declare association loading explicitly at query time [Recommended]

When associated data is needed, declare eager loading or a join in the specific query. Do not enable eager loading globally in entity mappings, or queries that do not need the associated data pay for it too.

**Source:** Use The Index, Luke

### Check the SQL the ORM actually generates [Recommended]

For important queries, look at the generated SQL and the execution plan, not just how the ORM call is written. For language-specific usage, see the relevant files under `languages/`.
