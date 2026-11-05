# Metrics and alerts

This document sets what to monitor, how to name metrics, and how to control cardinality and design alerts. For SLO definitions, see [Operational readiness](../reliability/operations.md). Naming examples follow Prometheus conventions.

**Applies to:** production service (all rules in this document)

## What to monitor

### Use RED for services, USE for resources [Recommended]

| Method | Target | Metrics |
|---|---|---|
| RED | Services and endpoints | Rate (request rate), Errors (error count), Duration (latency distribution) |
| USE | CPU, memory, disk, connection pools, thread pools | Utilization, Saturation, Errors (error count) |

RED answers "how is the user experience", and USE answers "which resource is the bottleneck". Saturation usually shows up as queue length.

**Source:** Grafana: RED Method, Gregg: USE Method, Google SRE: Monitoring

### Choose core metrics by service type [Recommended]

| Type | Core metrics |
|---|---|
| Online services | Request count, error count, latency (P95, P99), in-flight requests |
| Async processing | Items in and out, queue backlog, oldest message age, end-to-end duration |
| Batch jobs | Last success time, run duration, items processed |
| Resource pools | Connection pool utilization, waiter count, thread pool queue size |

**Source:** Prometheus: Instrumentation

### Pair failure metrics with total metrics [Recommended]

When recording failure counts, also have a total attempt count, or the failure rate cannot be calculated. Count requests when they finish, so the count stays consistent with errors and latency. Instrument online calls on both the caller and callee sides. When the two sides disagree, the problem is often the network or queueing.

**Source:** Prometheus: Instrumentation

### Look at distributions, not averages [Recommended]

Use histograms for latency, with exponentially distributed buckets that cover the SLO threshold. Do not use the summary type when you need cross-instance aggregation, because quantiles cannot be merged.

**Source:** Prometheus: Histograms, Google SRE: Monitoring

## Metric naming

### Follow uniform naming rules [Recommended]

| Rule | Avoid | Prefer |
|---|---|---|
| Include an application or domain prefix | `requests_total` | `payment_requests_total` |
| Use base units | `latency_ms` | `request_duration_seconds` |
| Put the unit in the name, in plural | `request_duration` | `request_duration_seconds` |
| End cumulative counters with `_total` | `http_requests` | `http_requests_total` |
| Use `_timestamp_seconds` for timestamps | `last_run` | `last_success_timestamp_seconds` |
| Express ratios from 0 to 1 | `disk_usage_percent` | `disk_usage_ratio` |

Always use bytes, not bits.

**Tooling:** `promtool check metrics` can check naming.

**Source:** Prometheus: Naming

### Express one quantity per metric [Recommended]

Do not mix units or meanings. Summing or averaging across all dimensions should make sense. Otherwise split it into several metrics.

**Source:** Prometheus: Naming

### Use labels for dimensions, not values in names [Recommended]

Metric names must not be generated dynamically, and label names are not repeated in metric names.

**Example:**

Avoid:

```text
http_responses_500_total
http_responses_403_total
```

Prefer:

```text
http_responses_total{code="500"}
http_responses_total{code="403"}
```

**Source:** Prometheus: Instrumentation, Prometheus: Naming

### Choose the right metric type and form [Recommended]

Use a Gauge when the value can decrease, and a Counter when it only increases. Do not take the rate of a Gauge. To record the last occurrence time, export a Unix timestamp, not a time difference. Export 0 first for predictable series, because series that appear only after an event cannot be calculated or alerted on normally.

**Source:** Prometheus: Instrumentation

### Choose one naming system per project [Recommended]

OpenTelemetry semantic conventions use dotted names without the `_total` suffix, unlike Prometheus style. Use one within a project, let the export layer convert, and do not mix them in code.

**Source:** OpenTelemetry: Naming, Prometheus: Naming

## Cardinality control

### Use only bounded enum values as labels [Required]

Every label combination is a time series that consumes memory, CPU, and storage. These values must not be labels:

- User IDs, email addresses, order numbers, request IDs, `traceId`
- Untemplated URL paths
- Raw error messages, timestamps

**Example:**

Avoid:

```text
http_requests_total{path="/orders/1001",user_id="42"}
```

Prefer:

```text
http_requests_total{route="/orders/{id}",method="GET",code="200"}
```

**Why:** Unbounded labels make the series count explode and overwhelm the monitoring system, often only during a failure.

**Source:** Prometheus: Naming, Prometheus: Instrumentation

### Control the cardinality of each metric [Recommended]

Most metrics need no labels. Keep the cardinality of a single metric within 10 where possible. Above 100, change approach, for example by reducing dimensions. When unsure, leave the label out and add it once there is a concrete use. To investigate by user or order, use logs and distributed tracing.

**Source:** Prometheus: Instrumentation

## Alert design

### Alert on symptoms users can perceive [Recommended]

Prefer alerting on symptoms such as high error rate and high latency, and set alerts as high in the call stack as possible. Cause signals such as high CPU are for investigation and do not page anyone directly.

**Why:** Cause-based alerts are numerous and noisy, and they always miss some causes.

**Source:** Prometheus: Alerting, Google SRE: Monitoring

### Make every page actionable [Recommended]

Use pages only for urgent situations that need human judgment. Automate what can be automated, and delete alerts that have no action. Alert content includes impact scope, a dashboard link, and a runbook link.

| Level | Condition | Notification |
|---|---|---|
| Page | Affecting or about to affect users, needs immediate handling | Phone call, instant page |
| Ticket | Needs handling, but can wait until working hours | Ticket |
| Dashboard | For investigation only | No notification |

**Why:** Noisy alerts cause fatigue, and real failures get ignored too.

**Source:** Google SRE: Monitoring, Prometheus: Alerting

### Use SLO-based burn rate alerts [Recommended]

Burn rate is how fast the error budget is consumed. A rate of 1 means the budget runs out exactly at the end of the window. Fast burn uses an hour-scale window and pages. Slow burn uses a day-scale window and opens a ticket. Use a long window and a short window together: the long window confirms the impact is large enough, the short window confirms the problem is still happening, and the alert clears quickly after a fix.

**Example:** With a 30-day window and a 99.9% target, to alert when 2% of the budget is consumed within 1 hour, the threshold is `0.02 × 720 ÷ 1 = 14.4`, meaning an error rate above 1.44%.

**Exceptions:** With very few requests, a single failure triggers a high burn rate. You can add synthetic probe traffic, combine related services into one calculation, or widen the window.

**Source:** Google SRE Workbook: Alerting

### Use different alerts for different service types [Recommended]

| Type | Alert condition |
|---|---|
| Online services | User-visible error rate and latency |
| Async processing | Time for data to pass through the system affects users |
| Batch jobs | No success for too long, with a threshold covering at least two full runs |
| Capacity | Near the limit, needs human intervention |

Alert separately on important low-traffic endpoints, or high-traffic endpoints will drown them out.

**Source:** Prometheus: Alerting

### Make alerts tolerate brief fluctuations [Recommended]

Set a duration so momentary spikes do not fire alerts.

**Example:**

```yaml
- alert: PaymentHighErrorRate
  expr: |
    sum(rate(payment_requests_total{code=~"5.."}[5m]))
      / sum(rate(payment_requests_total[5m])) > 0.05
  for: 5m
  labels:
    severity: page
  annotations:
    summary: Payment API error rate above 5%
    runbook: https://example.com/runbooks/payment-errors
```

**Source:** Prometheus: Alerting

### Monitor the monitoring system itself [Recommended]

Set up end-to-end probes for the alerting pipeline, and use external black-box probes as a fallback, so a total failure of internal monitoring is still detected.

**Source:** Prometheus: Alerting
