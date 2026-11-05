# Background jobs

This document covers scheduled jobs and job queues. For message broker delivery semantics and dead-letter configuration, see [messaging.md](messaging.md). For long-running tasks at the API layer, see [rest.md](../api/rest.md).

## Basic requirements

### Design jobs to run more than once [Required]

Running with the same input several times gives the same result. Repeated execution is normal: queues deliver at least once, schedules can overlap, and restarts replay half-finished work. For jobs that cannot be idempotent, such as sending email, state the trade-off that duplicates may occur.

**Example:**

Avoid: a job that grants points to all users on every run.

Prefer: recording the grant batch with a unique index, and skipping on repeat runs.

```sql
INSERT INTO point_grants (user_id, batch_date, points) VALUES (?, ?, ?)
ON CONFLICT (user_id, batch_date) DO NOTHING;
```

**Source:** Azure: Background Jobs, Kubernetes CronJob

### Persist jobs, not just in memory [Required]

Put jobs that must not be lost in a persistent queue or job table, not an in-process memory queue.

**Why:** When a process restarts or deploys, all jobs in memory are lost, with no record.

### Pass only identifiers as job parameters [Recommended]

Job parameters carry small data such as IDs. At execution time, read the latest state and confirm again that the job still needs to run.

**Example:**

Avoid:

```json
{ "type": "order.close", "order": { "id": "1001", "status": "pending", "items": [] } }
```

Prefer:

```json
{ "type": "order.close", "order_id": "1001" }
```

**Why:** Data may change while the job is queued, and a snapshot in the parameters leads to wrong decisions.

### Deploy background jobs separately from API services [Recommended]

Run heavy jobs in separate processes, so they can scale and release independently and do not take resources from API services. API services scale by request volume, and background jobs scale by queue depth.

**Source:** Azure: Background Jobs

## Scheduled jobs

Do not put scheduled jobs in a database scheduler. See [schema-design.md](../data/schema-design.md).

### Prevent duplicate runs in multi-instance deployments [Required]

Keep only one scheduling entry point, or use a database unique constraint to claim the right to this run. A job that runs longer than the schedule interval also overlaps with the next run, and this must be handled explicitly. For distributed lock caveats, see [consistency.md](../reliability/consistency.md).

**Example:**

Avoid: starting the same timer inside every service instance.

Prefer: claiming with the job name and scheduled time, so only the instance whose insert succeeds runs the job.

```sql
INSERT INTO job_runs (job_name, scheduled_at, status) VALUES (?, ?, 'running')
ON CONFLICT (job_name, scheduled_at) DO NOTHING;
```

In Kubernetes, use `CronJob`. The key settings are:

```yaml
spec:
  schedule: "*/10 * * * *"
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
```

`CronJob` can create a job twice or not at all, so the job itself must still be idempotent. `concurrencyPolicy` defaults to `Allow`, which permits concurrent runs. `Forbid` skips the overlapping run.

**Source:** Azure: Background Jobs, Kubernetes CronJob

### State the schedule time zone [Required]

Write the schedule time zone in configuration, default to UTC, and do not depend on the server's local time zone. Kubernetes uses `.spec.timeZone` and does not support `TZ` or `CRON_TZ` in `schedule`.

**Source:** Kubernetes CronJob

### Process data windows by scheduled time [Recommended]

The scheduled time, not the actual start time, decides what a job processes. A delayed start or a rerun then handles the same batch of data.

**Example:**

Avoid:

```sql
SELECT id FROM orders WHERE created_at >= NOW() - INTERVAL '1 day';
```

Prefer:

```sql
SELECT id FROM orders WHERE created_at >= ? AND created_at < ?;
```

## Job queues

### Claim safely when using a database as a job queue [Recommended]

For low job volumes, one job table is enough, and you do not need new middleware. When claiming, skip rows that are already locked, so workers do not wait on each other.

**Example:**

```sql
SELECT id, payload FROM jobs
WHERE attempts < ?
  AND ((status = 'pending' AND run_at <= ?)
    OR (status = 'running' AND lease_until <= ?))
ORDER BY run_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

UPDATE jobs SET status = 'running', lease_until = ?, attempts = attempts + 1 WHERE id IN (?);
```

The claim transaction does only three things: select jobs, write `running` and the lease expiry, and commit. The job runs outside the transaction.

The job table includes at least the state, next run time, retry count, and lease expiry. After a worker crashes, the claim condition picks up jobs whose lease has expired, incrementing the retry count on each claim. Jobs that reach the limit are no longer claimed and are treated as failed. The lease duration must be longer than the job's timeout.

### Use separate queues for different priorities [Recommended]

Use different queues and workers for slow batch jobs and short real-time jobs, so the former does not block the latter.

### Match consumption to downstream capacity [Recommended]

Set concurrency by what the database and downstream services can handle, not just by queue backlog.

**Why:** Blindly scaling consumers during a backlog shifts the pressure to the database or downstream services.

## Sharding

### Split big jobs into independently retryable shards [Recommended]

Split data into shards by primary key range or hash. Each shard runs, records state, and retries independently. When one shard fails, you do not need to rerun all the data.

| Method | Use for |
|---|---|
| Primary key range | Tables with roughly contiguous primary keys; simple to split |
| Hash modulo | Non-contiguous primary keys, or when even distribution is needed |
| By tenant | Tenants that do not affect each other; watch for skew from large tenants |

### Save progress in long jobs [Recommended]

Long or multi-step jobs periodically write progress to persistent storage and continue from the checkpoint after a restart. Commit bulk writes in batches according to [transactions.md](../data/transactions.md).

**Example:** When processing reaches primary key 50000, write `last_id = 50000`, and start next time from `id > 50000`.

**Source:** Azure: Background Jobs

## Retry and termination

Retry only transient failures. Mark permanent failures as failed right away. For the rules, see [resilience.md](../reliability/resilience.md) and [error-handling.md](../../general/error-handling.md).

### Give failed jobs a limit and a dead-letter path [Required]

**Applies to:** production service

Jobs that exhaust their retries go to a dead-letter queue or are marked failed, keeping the input and failure reason, with an alert configured. They can be resubmitted after a fix.

**Source:** Azure: Background Jobs

### Set an execution timeout on jobs [Required]

Every job type has a maximum execution time. On timeout, abort it and treat it as failed. External calls must set timeouts too.

**Why:** A stuck job raises no error, but it keeps holding a worker slot and a lease.

### Exit gracefully on a termination signal [Required]

**Applies to:** production service

After receiving `SIGTERM`, stop claiming new jobs, and exit after finishing the current job or saving progress. The platform's grace period must exceed the typical duration of a single work item, for example Kubernetes `terminationGracePeriodSeconds`.

**Source:** Azure: Background Jobs

## Observability

### Record the full job lifecycle [Recommended]

**Applies to:** production service

Record start, completion, and failure, including the job type, correlation ID, and duration, and save the result. If only the start is recorded, stuck or crashed jobs look like they are running normally. Multi-step jobs pass the correlation ID throughout. See [tracing.md](../observability/tracing.md).

**Source:** Azure: Background Jobs

### Monitor for jobs that did not run [Recommended]

**Applies to:** production service

Compare actual runs against the expected schedule, and alert when one is missing. A job that never fires produces no error, so error logs alone will not reveal it.

**Source:** Azure: Background Jobs

### Track queue wait time [Recommended]

Besides execution time, also track total time from enqueue to completion, queue depth, and dead-letter count. Total time is what the business experiences. For metrics and alerts, see [metrics-and-alerts.md](../observability/metrics-and-alerts.md).

**Source:** Azure: Background Jobs
