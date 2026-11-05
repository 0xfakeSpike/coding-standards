# Backend performance

This document covers performance techniques specific to the backend. For the requirements to measure first, compare before and after, and report numbers, see [Performance methodology](../../general/performance.md). For SQL and index optimization, see [Indexes and queries](../data/indexes-and-queries.md). For caching, see [Cache and Redis](../data/cache-and-redis.md).

## Profiling

### Locate the component first, then the code [Recommended]

First use existing traces, metrics, or logs to find which service or dependency is slow, then profile that process. Do not change code on a hunch.

### Distinguish CPU time from wait time [Recommended]

| Symptom | Analysis |
|---|---|
| High CPU | Sampling profiler, use a flame graph to look at the widest call stacks |
| CPU not high but requests are slow | Analyze wait time, such as I/O, locks, and connection pools |

**Why:** CPU profiling only shows threads while they are running. Most slow backend requests are waiting, so a CPU flame graph alone will not find the cause.

**Source:** Gregg: Flame Graphs, Gregg: Off-CPU Analysis

## Connection pools

### Reuse connections [Recommended]

Use connection pools for databases, caches, and HTTP clients. Do not create a new connection for each request. Create client objects once, globally.

### Bigger connection pools are not always better [Recommended]

Determine pool size from load tests. You can start around twice the database's CPU core count, then adjust from measurements.

**Why:** Once connections exceed what the database can handle, context switching and lock contention slow down every request. Shrinking the pool often lowers latency at the same time.

**Example:**

```json
{ "pool": { "maxSize": 20, "acquireTimeoutMs": 500 } }
```

**Source:** HikariCP Docs

### Keep total connections across all instances under the limit [Required]

**Applies to:** production service

Per-instance pool size times maximum instance count must be below the database's connection limit, with headroom left for operations tools. Calculate this number before scaling out.

### Set a timeout on acquiring a connection [Required]

**Applies to:** production service

Fail fast when no connection is available. Do not queue indefinitely. The acquire timeout counts toward the request's time budget. See [Resilience](resilience.md). Monitor connections in use, waiters, and wait time.

### Hold connections briefly [Required]

**Applies to:** production service

Do not call external services or do slow computation while holding a connection. A request that holds several connections at once can easily deadlock the pool. Eliminate this in the code first.

**Source:** HikariCP Docs

## Batching

For partial failure semantics of batch endpoints, see [rest.md](../api/rest.md). When processing items one by one, make it possible to retry only the failed items.

### Eliminate remote calls in loops [Recommended]

**Applies to:** production service

Replace queries and calls inside loops with batch operations. Implicit loading in an ORM also causes N+1.

**Example:**

Avoid:

```sql
SELECT id, sku_id FROM order_items WHERE order_id = 1;
SELECT id, sku_id FROM order_items WHERE order_id = 2;
SELECT id, sku_id FROM order_items WHERE order_id = 3;
```

Prefer:

```sql
SELECT id, sku_id FROM order_items WHERE order_id IN (1, 2, 3);
```

**Source:** Azure: Chatty I/O

### Cap batch size [Required]

**Applies to:** production service

Limit batch requests by item count and bytes, process large datasets in batches, and keep the send rate within what the downstream can handle. Read only the fields and rows you actually need.

**Why:** Oversized batches cause long transactions, memory spikes, and timeouts.

**Source:** Azure: Chatty I/O, Azure: Rate Limiting

## Load testing

**Applies to:** production service (all rules in this section)

### Test past the breaking point [Recommended]

Push until the system has problems, then keep adding load. Under overload, throughput should stay steady, not fall to zero. Assume a service that has not been tested past its breaking point will fail in the worst possible way.

**Source:** AWS: Load Shedding, Google SRE: Cascading Failures

### Generate load at a constant rate [Recommended]

When measuring latency, use a tool with a fixed arrival rate. Closed-loop tools slow down automatically when the service slows, undercounting the slowest requests. This is called coordinated omission, and it makes P99 look much better than it is.

**Tooling:** The `-R` option of wrk2 sets a constant throughput.

**Source:** wrk2 Docs

### Keep environment and data close to production [Recommended]

Keep data volume, data distribution, and dependencies close to production. Query plans on small tables differ from production, and uniform request parameters inflate cache hit rates. Also measure latency and success rate from the client's perspective.

**Source:** AWS: Load Shedding

## Memory

### Stream large data [Required]

**Applies to:** production service

Use streaming reads and writes or pagination for files, exports, and large result sets. Do not load them into memory all at once.

### Cap in-memory collections and individual requests [Required]

**Applies to:** production service

In-process caches, queues, and accumulating lists all need a capacity limit and an eviction policy. Also limit request body size, page size, and batch item count. Otherwise, a single request can exhaust memory.

### Leave headroom in memory limits [Recommended]

Set the container memory limit above the steady peak, and leave headroom for the runtime's non-heap memory. A process that exceeds the limit is killed outright. See [Containers](../../general/containers.md). Leaks often do not show up in short tests, so use soak tests to watch the memory curve. For runtime tuning in each language, see the language layer.

**Source:** k6 Test Types

## P99 management

Use histograms to aggregate percentiles across instances. See [Metrics and alerts](../observability/metrics-and-alerts.md).

### Measure success and failure latency separately [Recommended]

Fast-failing requests pull down latency statistics. Keep requests rejected by rate limiting out of normal request latency.

**Source:** AWS: Load Shedding

### Watch latency at high utilization [Recommended]

As utilization rises, queuing, lock contention, and garbage collection all get worse, and latency degrades before throughput does. Verify P99 targets at the target utilization, not when idle.

**Source:** AWS: Load Shedding

### Watch for fan-out amplification [Recommended]

When a request calls multiple downstreams in parallel, total latency depends on the slowest one. When calling 100 downstreams, even if each has only 1% slow requests, about 63% of overall requests are slowed down. Reduce fan-out, and set a deadline for each downstream.

### Break down slow request time [Recommended]

Use distributed tracing to look separately at queuing, processing, and downstream wait time. A bimodal latency distribution usually means some requests are stuck until timeout. The average is meaningless in that case, so look at the distribution.

**Source:** Google SRE: Cascading Failures

### Fix tail latency at the root cause [Recommended]

First find what is slow, such as lock contention, garbage collection, or slow queries. Do not mask it by raising timeouts or adding retries.
