# Resilience

This document defines how a service protects itself when dependencies slow down, fail, or overload. For language-independent error principles, see [Error handling principles](../../general/error-handling.md).

## Trade-offs

### Do not add compensation for rare failures [Required]

Do not add complexity for corner cases. For rare failures with acceptable consequences, when the benefit does not cover the long-term maintenance cost, do not add fallback paths or scheduled compensation. Assess cases involving money or data loss separately, and explain the trade-off to the user.

**Source:** Team

## Timeout budgets

### Set timeouts on all network calls [Required]

Set both a connect timeout and a request timeout, including for databases, caches, and cross-process calls on the same machine. Do not use a client library's default of waiting forever.

**Why:** A call without a timeout holds threads, connections, and memory indefinitely, and eventually takes the service down.

**Source:** AWS: Timeouts and Retries

### Choose timeouts from downstream latency [Recommended]

First decide the acceptable false timeout rate, then take the matching downstream latency percentile. For example, if you accept 0.1% false timeouts, use P99.9 as the reference. For calls across the public internet, add the worst-case network latency. Leave margin when P99.9 is close to P50. Also confirm whether the timeout includes DNS resolution, the TLS handshake, and connection setup.

**Why:** A timeout that is too long gives no protection. One that is too short makes many requests retry, and a small slowdown can grow into an outage.

**Source:** AWS: Timeouts and Retries

### Share one time budget across the call chain [Recommended]

For cross-service calls, the entry point sets the total deadline and passes the remaining time downstream. Each hop's timeout does not exceed the remaining time. When the server queues requests, check the remaining time after dequeuing and drop expired requests directly. A single-process service only needs a timeout on each call.

**Why:** When each layer sets its timeout independently, downstream keeps working after upstream has given up, and those resources are all wasted.

**Example:**

| Call | Avoid | Prefer |
|---|---|---|
| Entry point | 2 s | 2 s |
| Service A calls service B | 5 s | Remaining time minus A's subsequent processing time |
| Service B calls the database | 10 s | Remaining time received minus B's subsequent processing time |

**Exceptions:** When clocks differ between machines, passing a duration is more reliable than passing an absolute time. Subtract queuing and transmission time from the duration.

**Source:** Google SRE: Cascading Failures, AWS: Load Shedding

## Retries and backoff

### Retry only temporary failures and operations safe to repeat [Required]

Both conditions must hold: the failure is temporary, and repeating the operation has no side effects or the operation is already idempotent. Otherwise, do not retry.

A timeout does not mean the other side did not execute. Make operations with side effects idempotent first. See [Consistency](consistency.md).

Every "Yes" in the table below assumes the operation is idempotent. Only connection failures that definitely confirm the request did not reach the server do not require idempotency.

| Failure | Retry? |
|---|---|
| Connection failure, network interruption | Yes |
| 503, 429 | Yes, respect `Retry-After` |
| 502, 504 | Yes |
| Timeout | Yes |
| 500 | No by default, yes when you can confirm the failure is temporary |
| Other 4xx | No |
| Business rejection, program defect | No |

**Why:** Retrying permanent errors only adds load. Retrying non-idempotent operations produces side effects such as duplicate charges.

**Source:** AWS: Timeouts and Retries, Azure: Retry Storm

### Use capped exponential backoff with jitter [Required]

Limit the number of retries. Grow the wait time exponentially up to a cap, then add random jitter. Total retry time does not exceed the caller's time budget. Stagger the trigger times of scheduled tasks and periodic requests as well.

**Why:** Without backoff, retries add pressure on the downstream. Without jitter, all clients retry at the same moment and create a new peak.

**Example:**

```json
{
  "retry": {
    "maxAttempts": 3,
    "baseDelayMs": 100,
    "maxDelayMs": 2000,
    "jitter": "full"
  }
}
```

**Source:** AWS: Timeouts and Retries, Azure: Retry Storm

### Retry at one layer only [Required]

**Applies to:** production service

Pick only one layer in the call chain to own retries, usually the layer closest to the failure that can determine the error type. Queue redelivery and built-in SDK retries each count as a layer.

**Why:** Retries multiply layer by layer. With five layers each trying three times, load on the bottom layer can reach 243 times.

**Source:** AWS: Timeouts and Retries, Google SRE: Cascading Failures

### Limit total retries with a retry budget [Recommended]

Beyond the per-request limit, high-traffic clients also limit retries as a share of total requests, or limit the retry rate with a token bucket. When a dependency is unhealthy, retries should decrease automatically.

**Source:** AWS: Timeouts and Retries, Google SRE: Handling Overload

## Circuit breakers

### Configure circuit breakers on critical dependencies as needed [Recommended]

Circuit breakers are for sustained failures, not transient errors. Prefer the implementation in your framework or service mesh.

| State | Behavior | Transition |
|---|---|---|
| Closed | Let requests through, count recent failures | Failures within the time window reach the threshold: move to open |
| Open | Fail requests immediately | Wait time elapses: move to half-open |
| Half-open | Let a few probe requests through | Consecutive successes: move to closed. Failure: move back to open |

**Exceptions:** Circuit breakers are not needed when accessing in-process resources, when the platform already handles failure recovery, or when the flow is message-driven with a dead-letter queue. Do not add a circuit breaker when a simple retry budget is enough. The mode switching it introduces is hard to test and can lengthen recovery.

**Source:** Azure: Circuit Breaker, AWS: Timeouts and Retries

### Coordinate circuit breakers with other mechanisms [Recommended]

- Retry logic recognizes errors returned by the circuit breaker and stops retrying.
- Keep downstream timeouts reasonable. If a timeout is too long, threads fill up before the circuit opens.
- Callers handle circuit breaker errors, for example by degrading or reporting an explicit error.
- Expose state changes as metrics, and allow forcing the circuit open or closed manually.
- When the downstream has multiple independent shards, use a separate circuit breaker for each. Otherwise, one shard's failure blocks the others.

**Source:** Azure: Circuit Breaker

## Isolation

### Isolate resources per dependency [Recommended]

When calling multiple downstreams, give each its own connection pool or concurrency limit, and use separate resource pools for critical and non-critical business. When container resource limits, gateway rate limiting, and separate queues already provide isolation, do not rewrite it in application code. Monitor each partition separately.

**Why:** With a shared resource pool, one slow dependency uses up all the connections, and requests to other dependencies fail too.

**Exceptions:** Do not isolate when resource utilization must not drop, or when the system is too small for isolation to pay off.

**Source:** Azure: Bulkhead

## Rate limiting

### Set limits on the resource that saturates first [Recommended]

**Applies to:** production service

Find the bottleneck first, then choose the limiting dimension. When the bottleneck is concurrency or downstream capacity, limiting by request rate does not protect the system. When operations differ widely in cost, meter by cost. For whether to make thresholds dynamic configuration, see [Configuration](../architecture/configuration.md).

**Source:** Azure: Throttling, Google SRE: Handling Overload

### Limit per caller and choose the algorithm deliberately [Recommended]

Set quotas per tenant or caller so a single caller cannot use up the resources.

| Algorithm | Characteristics | Use when |
|---|---|---|
| Token bucket | Allows some bursting | Gateways absorbing short peaks |
| Leaky bucket | Releases at a steady rate | Backends needing a steady inflow |
| Fixed window | Simple to implement, consecutive bursts possible at window boundaries | Low precision is acceptable |
| Sliding window | Smoother, more state | Precise limits are needed |

When multiple instances share a quota, local counting is fast but imprecise, and central counting is precise but adds latency. For endpoints that need central counting, see [Audit and abuse prevention](../security/audit-and-abuse.md).

**Source:** Azure: Throttling, Google SRE: Handling Overload

### Make rejections explicit and cheap [Recommended]

**Applies to:** production service

Return 429 when a caller exceeds its quota and 503 when the whole service is overloaded, with `Retry-After` where possible. Put coarse global and per-IP rate limiting before authentication and parsing. Put rate limiting per user, account, or tenant after identity resolution.

**Why:** When rejecting costs nearly as much as normal processing, rejected requests can still bring the service down.

**Source:** Azure: Throttling, AWS: Load Shedding

### Throttle yourself when calling rate-limited services [Recommended]

When a batch task calls a downstream with a quota, control the send rate yourself. That is more efficient than sending everything and then handling a flood of rejections.

**Source:** Azure: Rate Limiting

## Graceful degradation

### Protect core requests first under overload [Recommended]

Near overload, actively reject some requests so accepted requests keep normal latency. When priorities can be distinguished, reject low-value requests first, and let downstream calls inherit the upstream priority.

**Why:** Under overload, throughput does not hold steady, it drops. Accepting everything means everything times out.

**Source:** AWS: Load Shedding, Google SRE: Handling Overload

### Keep degradation paths simple and tested [Recommended]

**Applies to:** production service

Degradation should reduce work, for example by hiding the recommendations module or returning cached results. Do not switch to another complex path.

Rarely run degradation paths must be tested regularly.

**Why:** Paths that do not normally run hide defects most easily, and they often first trigger at the worst moment, which can amplify the outage.

**Source:** AWS: Fallback Strategies, Google SRE: Cascading Failures

### Do not disguise degraded results as normal results [Required]

Callers must be able to tell that a result is incomplete. Do not rewrite a dependency failure into an empty data response.

## Backpressure

### Bound all queues and buffers [Required]

**Applies to:** production service

Limit the length of in-process queues, thread pool queues, and batch buffers, and reject when they are full. Keep queues short relative to the thread pool.

**Why:** An unbounded queue only turns overload into latency and memory growth. By the time a request reaches the front, the caller has long given up.

**Source:** Google SRE: Cascading Failures

### Pass overload signals upstream [Recommended]

**Applies to:** production service

When a downstream returns 429 or 503 and retries are exhausted, pass that upstream as is. Do not rewrite it to 500, so the caller knows to slow down.

**Source:** Azure: Throttling

### Control the consumption rate in async pipelines [Recommended]

Queues can only level load. They cannot fix a long-term shortfall. Monitor queue depth and the age of the oldest message. When the average production rate stays above the consumption rate, scale out or reduce volume at the producer. When scaling out consumers, limit the total rate to the downstream.

For rules about the messages themselves, see [Messaging](../async/messaging.md).

**Source:** Azure: Queue-Based Load Leveling, AWS: Queue Backlogs
