# Distributed tracing

This document sets distributed tracing integration, context propagation, Span conventions, and sampling. The rules are based on OpenTelemetry. For language-specific integration, see each language's docs.

**Applies to:** production service (all rules in this document)

## Integration

### Use OpenTelemetry [Recommended]

New services produce trace data with OpenTelemetry and do not bind to a vendor's SDK. For frameworks, HTTP clients, and database drivers, prefer existing auto-instrumentation, then add manual instrumentation for key business steps.

### Use a uniform service identity [Recommended]

Set `service.name` on every service, matching the service name in logs and metrics. Only then can the three signals be correlated by source.

**Source:** OpenTelemetry: Logs Specification

### Follow semantic conventions [Recommended]

For common operations such as HTTP, database, and messaging, use the attribute names in the semantic conventions. Do not invent your own.

**Why:** Only with uniform names can data from different services and libraries be queried together.

**Source:** OpenTelemetry: Semantic Conventions

## Context propagation

### Pass context across service calls [Recommended]

Carry `traceId` through the whole call chain. Use the W3C Trace Context `traceparent` and `tracestate` headers by default. Every intermediate layer needs to forward them, or the trace breaks.

**Example:**

```http
GET /orders/1001 HTTP/1.1
Host: order.internal
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

**Source:** OpenTelemetry: Context Propagation, W3C Trace Context

### Do not pass identifiers by hand within a process [Recommended]

Pass context through the language's context mechanism. Do not pass `traceId` in business function parameters. When switching across thread pools and async tasks, confirm the context is not lost.

### Carry context in async messages too [Recommended]

Write context into message properties, not the business payload. Producers inject it, and consumers extract it.

Consumer-side Spans relate to the message's creation context through a Link. In batch consumption, one Span covers several messages, which only links can express.

**Source:** OpenTelemetry: Messaging Spans

### Do not trust externally supplied context [Required]

`traceparent` and Baggage from the public internet may be forged. Start a new trace at the entry point, and save the context sent by the browser as a link, not as a trusted parent. Trust only values passed by internal callers.

**Why:** Forged headers can pollute trace data or exploit parsing flaws.

**Source:** OpenTelemetry: Context Propagation, W3C Trace Context

### Do not send internal context to external systems [Recommended]

When calling third parties, `traceId` and Baggage may expose internal structure. Assess whether to send them.

**Source:** OpenTelemetry: Context Propagation

### Keep sensitive data out of Baggage [Required]

Baggage is passed to all downstream services and may be recorded. Do not put credentials, API keys, or personal information in it. Do not put personally identifiable information in `tracestate` either.

**Source:** OpenTelemetry: Context Propagation, W3C Trace Context

## Span conventions

### Keep Span names low-cardinality [Recommended]

Name Spans by the category of operation, with no variable values such as IDs. Put variable values in attributes.

**Example:**

| Name | Verdict |
|---|---|
| `get` | Avoid: too generic |
| `GET /orders/1001` | Avoid: contains a specific ID |
| `GET /orders/{id}` | Prefer |
| `order.reserve_stock` | Prefer |

HTTP Spans use `{method} {route}`. When there is no low-cardinality route, use only `{method}`, not the raw path.

**Source:** OpenTelemetry Trace API, OpenTelemetry HTTP Span

### Create Spans only for meaningful operations [Recommended]

Create Spans for cross-process calls, database operations, and important business steps. Do not create one for every function.

**Why:** Too many Spans add overhead and make traces hard to read.

### Set status according to semantic conventions [Recommended]

| Situation | Status |
|---|---|
| Operation succeeded | Leave as `Unset` |
| Operation failed | `Error`, and set `error.type` |
| HTTP server returns 4xx | `Unset` |
| HTTP server returns 5xx | `Error` |
| HTTP client receives 4xx or 5xx | `Error` |
| Caller cancels deliberately | `Unset` |

Use `Ok` only when the application explicitly determines success, because it overrides other statuses.

**Source:** OpenTelemetry Trace API, OpenTelemetry HTTP Span

### Do not mark handled errors as failures [Recommended]

When an operation succeeds after retries, or the error has been properly handled, the Span describing that operation does not record an error.

**Source:** OpenTelemetry: Recording Errors

### Use namespaces for custom attributes [Recommended]

Attribute names are lowercase, with dots separating namespaces and underscores between words, for example `order.item_count`. Do not occupy `otel.` or namespaces of existing semantic conventions. Company or application custom attributes carry their own prefix.

**Source:** OpenTelemetry: Naming

### Keep sensitive data out of Span attributes [Required]

Span attributes and events follow the same redaction rules as logs. See [Logging](logging.md). Do not record request bodies, SQL parameter values, or full URLs with query strings.

## Sampling

### Decide whether sampling is needed first [Recommended]

With very small data volumes, or when you look only at aggregates, sampling is not needed. Do not sample when regulations forbid losing data. Consider it only when traffic is high and most requests are normal.

**Source:** OpenTelemetry: Sampling

### Choose where to sample by need [Recommended]

| Method | Decision point | Pros | Cons |
|---|---|---|---|
| Head sampling | At the start of the request | Simple, low overhead | Cannot guarantee error traces are kept |
| Tail sampling | After the full trace is collected | Can keep errors and slow requests | Stateful component, high operating cost |

Start with head sampling by default. Add tail sampling only when you truly need to keep errors and slow requests.

**Source:** OpenTelemetry: Sampling

### Keep the sampling decision consistent across a trace [Recommended]

Sample deterministically based on `traceId`, and have downstream services follow the upstream decision, or traces will be missing pieces.

**Source:** OpenTelemetry: Sampling

### Put the sampling rate in configuration [Recommended]

Put the sampling rate in configuration, adjustable at runtime. See [Configuration](../architecture/configuration.md).

### Do not derive metrics from sampled traces [Recommended]

Request volume and error rate come from metrics, counted before sampling. Traces are for locating individual requests. See [Metrics and alerts](metrics-and-alerts.md).
