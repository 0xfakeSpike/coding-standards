# Transactions and locks

Focuses on MySQL InnoDB and PostgreSQL. For idempotency, distributed locks, Outbox, and Saga, see [consistency.md](../reliability/consistency.md).

## Transaction boundaries

### Keep transactions as short as possible [Required]

Put only the writes that must succeed together in a transaction, and commit as soon as they finish.

**Why:** The longer a transaction, the longer it holds locks, and the higher the chance of conflicts and deadlocks.

**Source:** MySQL Manual, PostgreSQL Docs

### Do no external operations inside a transaction [Required]

Inside a transaction, do not make network calls, process files, or wait for user actions.

**Why:** External calls extend lock hold time and cannot be rolled back with the transaction. When you need to write to the database and send a message together, see Outbox in [consistency.md](../reliability/consistency.md).

**Example:**

Avoid: open a transaction, update the order, call the payment API, wait for the response, commit.

Prefer: transaction 1 writes the `pending` state and the idempotency key, then commits. Outside any transaction, call the payment API with the same idempotency key. Transaction 2 writes the result with a conditional update. Treat a call timeout as unknown state: keep the order "processing", and retry or query with the same idempotency key. Whether to add a scheduled reconciliation job is the user's decision. Do not add one by default.

## Isolation levels

### State the isolation level in use [Recommended]

The two databases differ in default level and behavior. Confirm each item when migrating databases or relying on snapshot semantics.

| Item | MySQL InnoDB | PostgreSQL |
|---|---|---|
| Default level | `REPEATABLE READ` | `READ COMMITTED` |
| Plain reads at the default level | The transaction keeps using the snapshot from its first read | Each statement gets a new snapshot |
| Range locking at the default level | Uses gap locks and next-key locks to block inserts | Does not use gap locks |
| `READ COMMITTED` | No gap locks, so phantom reads can occur | Same as left; two reads in one transaction may return different results |
| Write conflicts under `REPEATABLE READ` | Waits for the lock, then continues | Reports a serialization failure and needs a retry |

**Source:** MySQL Manual, PostgreSQL Docs

### Retry the whole transaction on serialization failure [Required]

With PostgreSQL `REPEATABLE READ` or `SERIALIZABLE`, concurrent conflicts make the transaction fail with error code `40001`. The application must retry the whole transaction from the start, not just the last statement.

**Source:** PostgreSQL Docs

## Locks

### Use indexes for updates and locking [Required]

**Applies to:** production service

Conditions on `UPDATE`, `DELETE`, and `SELECT ... FOR UPDATE` must hit a suitable index, ideally a unique one.

**Why:** InnoDB row locks are actually placed on the index records scanned. Without a suitable index, the scan range grows, and more rows and gaps get locked.

**Example:**

Avoid: the `note` column has no index, so the statement scans and locks many records.

```sql
UPDATE orders SET status = 'closed' WHERE note = 'expired';
```

Prefer:

```sql
UPDATE orders SET status = 'closed' WHERE id = ?;
```

**Source:** MySQL Manual

### Do not lock unless you need to [Recommended]

When reading an old snapshot is acceptable, do not write `FOR UPDATE` or `FOR SHARE`. When a lock is truly needed, use the weakest lock that meets the need. In PostgreSQL, when key columns are not changed, `FOR NO KEY UPDATE` conflicts less than `FOR UPDATE`.

**Source:** MySQL Manual, PostgreSQL Docs

### Cap lock waits [Recommended]

PostgreSQL waits for locks indefinitely by default. Set a cap with `lock_timeout`, configured per role or session, not in global configuration. MySQL controls row lock waits with `innodb_lock_wait_timeout`.

**Source:** PostgreSQL Docs

## Deadlocks

### Lock in a fixed order for multi-row updates [Required]

**Applies to:** production service

All transactions access tables and rows in the same order, for example ascending primary key. Transactions then queue instead of waiting on each other.

**Example:**

Avoid: always updating the source account first in a transfer, so two transfers in opposite directions wait on each other.

Prefer: always updating the account with the smaller ID first, regardless of transfer direction.

```sql
SELECT id, balance FROM accounts WHERE id IN (?, ?) ORDER BY id FOR UPDATE;
```

**Source:** MySQL Manual, PostgreSQL Docs

### Retry deadlock errors a limited number of times [Recommended]

The database rolls back one of the transactions. The rolled-back transaction can be retried as a whole, with a cap on attempts. The MySQL error code is 1213, and the PostgreSQL SQLSTATE is `40P01`. For retry rules, see [resilience.md](../reliability/resilience.md).

**Why:** A fixed lock order only lowers the chance of deadlock and cannot eliminate it. On retry, the transaction must have no irreversible external side effects, which is one reason to do no external operations inside a transaction.

**Tooling:** In MySQL, use `SHOW ENGINE INNODB STATUS` to view the latest deadlock. When deadlocks are frequent, temporarily enable `innodb_print_all_deadlocks`.

**Source:** MySQL Manual, PostgreSQL Docs

## Long transactions

### Monitor and limit long transactions [Recommended]

**Applies to:** production service

A long transaction affects more than itself:

- In MySQL, it prevents purging of old row versions produced by other transactions, and other reads must also rebuild old versions.
- In PostgreSQL, it prevents `VACUUM` from reclaiming dead tuples, causing table bloat.
- When it holds a metadata lock, it blocks schema changes. See [migrations.md](migrations.md).

**Tooling:** In MySQL, check `information_schema.INNODB_TRX`. In PostgreSQL, check `pg_stat_activity`, and set `idle_in_transaction_session_timeout` for application roles.

**Source:** MySQL Manual, PostgreSQL Docs

### Batch large updates and deletes [Required]

**Applies to:** production service

Split into batches by primary key range or fixed row count, commit each batch separately, and leave a gap between batches.

**Why:** A single large statement causes a long transaction, lock waits, and primary/replica lag. Rollback on failure can take several times as long as execution, and killing the process does not cancel the rollback.

**Example:**

Avoid:

```sql
DELETE FROM event_logs WHERE created_at < '2026-01-01';
```

Prefer (MySQL), repeated until the affected row count is 0:

```sql
DELETE FROM event_logs WHERE created_at < '2026-01-01' LIMIT 5000;
```

Prefer (PostgreSQL, where `DELETE` does not support `LIMIT`):

```sql
DELETE FROM event_logs
WHERE id IN (
  SELECT id FROM event_logs WHERE created_at < '2026-01-01' LIMIT 5000
);
```

**Source:** MySQL Manual

## Optimistic locking

### Use optimistic locks or conditional updates for concurrent changes [Recommended]

When conflicts are rare, use a version column or a business condition instead of holding locks for a long time, combining the check and the write into one statement.

**Example:**

Version column:

```sql
UPDATE orders SET status = 'paid', version = version + 1
WHERE id = ? AND version = ?;
```

Business condition, which keeps inventory from overselling:

```sql
UPDATE inventory SET stock = stock - 1
WHERE sku_id = ? AND stock >= 1;
```

### Check the affected row count of conditional updates [Required]

An affected row count of 0 means a version conflict or an unmet condition. Handle it as an explicit outcome: re-read and retry, or return a failure to the caller. Do not treat it as success.

**Why:** Without the check, a lost update produces no error at all.
