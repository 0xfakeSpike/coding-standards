# Operational readiness

This document defines the operational capabilities a service must have before it ships. For baseline Kubernetes probe and resource configuration rules, see [Containers](../../general/containers.md). For releases, see [Release and rollback](../../general/release.md).

**Applies to:** production service (all rules in this document)

## Health checks

For the purpose of liveness and readiness probes, and the requirement not to depend on external services or shared dependencies, see [Containers](../../general/containers.md).

### Keep health checks lightweight [Recommended]

Health checks do not run expensive operations and must still respond promptly under overload. A slow health check gets instances removed by mistake and shifts load onto other instances. Check endpoints do not expose internal details.

**Source:** AWS: Health Checks

## Graceful startup and shutdown

### Accept traffic only when ready [Required]

The readiness check returns success only after connections are established, configuration is loaded, and warm-up is done. Exit immediately when a required dependency fails to initialize. Keep startup within a few seconds where possible. When startup is slow, use a startup probe instead of loosening the liveness probe.

**Why:** Cold instances perform poorly, and an instance with an empty cache can be overwhelmed by the traffic that just arrived.

**Source:** AWS: Timeouts and Retries, Google SRE: Cascading Failures, Twelve-Factor App, Kubernetes Probes

### Shut down gracefully on SIGTERM [Required]

| Order | Action |
|---|---|
| 1 | Fail the readiness check |
| 2 | Wait briefly for the load balancer to remove the instance |
| 3 | Stop accepting new requests and finish in-flight requests |
| 4 | Return background tasks to the queue or release locks |
| 5 | Close connection pools, flush logs and metrics, then exit |

**Why:** In Kubernetes, endpoint removal and SIGTERM happen at the same time and do not wait for each other, so new requests may still arrive after the signal.

**Source:** Kubernetes Pod Lifecycle, Twelve-Factor App

### Exit within the grace period [Required]

`terminationGracePeriodSeconds` defaults to 30 seconds and includes the time spent in `preStop`. Past that, the process is killed with SIGKILL. Keep the application's shutdown timeout shorter than the grace period, and keep the grace period longer than the longest request handling time.

The signal goes to PID 1 in the container. When a shell wraps the start command, confirm the signal reaches the application.

**Example:**

```yaml
terminationGracePeriodSeconds: 45
containers:
  - name: api
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]
```

**Source:** Kubernetes Pod Lifecycle

### Survive sudden death [Required]

Graceful shutdown is not guaranteed. A process can be killed at any time. Make long tasks reentrant or idempotent, and let a queue or lease mechanism hand them to another instance to continue.

**Source:** Twelve-Factor App

## Statelessness

In-process timers run on every instance. For scheduled job rules, see [Background jobs](../async/background-jobs.md).

### Keep services stateless [Recommended]

Put sessions, uploaded files, and task state in external storage so you can scale horizontally and replace instances at any time. Use memory and local disk only for temporary data within a single request. Put sessions in external storage with an expiry time. Do not use sticky sessions to pin users to an instance.

**Source:** Twelve-Factor App

## SLO

### Define SLIs from the user's perspective [Recommended]

SLIs measure the quality users actually experience. Do not pick internal metrics because they are "easy to measure". Keep the number of metrics small, and keep only those that can affect decisions.

| Service type | Common SLIs |
|---|---|
| Request/response | Success rate, latency percentiles |
| Data pipeline | Data freshness, processing correctness |
| Storage | Availability, durability |

**Example:** Over 30 days, at least 99.9% of responses succeed, and 99% of requests complete within 300 ms.

**Source:** Google SRE: Service Level Objectives

### Do not set the target at 100% [Recommended]

Set targets from user needs and cost, not from current performance. Start with a loose target, then tighten it based on experience. Connect SLOs to alerts. See [Metrics and alerts](../observability/metrics-and-alerts.md).

**Why:** An absolute target is unachievable, and it also blocks all changes.

**Source:** Google SRE: Service Level Objectives

## Capacity planning

### Use load tests to determine per-instance capacity [Recommended]

Use load tests to convert resource counts into service capacity, and redo them regularly. For endpoints with very different costs, calculate from actual resource consumption, not from request counts. Cover database connections, downstream quotas, and queue throughput in planning, not just application instances. For the method, see [Performance](performance.md).

**Source:** Google SRE: Introduction, Google SRE: Handling Overload, Azure: Capacity Planning

### Leave capacity headroom for failures [Recommended]

Capacity must withstand the loss of one availability zone or some instances. Autoscaling has a delay, so do not rely on it alone for traffic bursts.

**Source:** AWS: Load Shedding, Azure: Throttling
