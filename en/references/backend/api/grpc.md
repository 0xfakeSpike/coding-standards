# gRPC standards

This document covers conventions for Protobuf API definitions and gRPC calls. For language-specific usage, see the language layer. For general strategies on timeout budgets and retry backoff, see [resilience.md](../reliability/resilience.md).

## Proto style

### Follow the official naming style [Recommended]

| Item | Style | Example |
|---|---|---|
| File | `lower_snake_case.proto` | `order_service.proto` |
| Package | Dot-separated lowercase names, no reverse domain | `shop.orders` |
| Message, service, method | `TitleCase` | `CreateOrderRequest` |
| Field, oneof | `lower_snake_case` | `order_id` |
| repeated field | Plural | `items` |
| Enum type | `TitleCase` | `OrderStatus` |
| Enum value | `UPPER_SNAKE_CASE` | `ORDER_STATUS_PAID` |

An underscore must be followed by a letter and must not be the first or last character of a name. Avoid the prefixes `has_`, `get_`, `set_`, and `clear_` and the suffix `_value` in field names. They can conflict with generated code.

**Tooling:** `buf lint`

**Source:** Protobuf Style Guide

### Give enums a zero value and a type prefix [Required]

The first value is 0, ends in `_UNSPECIFIED` or `_UNKNOWN`, and carries no business meaning. Other values use the enum name as a prefix.

**Example:**

Avoid:

```proto
enum OrderStatus {
  PAID = 0;
  CANCELLED = 1;
}
```

Prefer:

```proto
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PAID = 1;
  ORDER_STATUS_CANCELLED = 2;
}
```

**Why:** Enum values are not scoped to the enum itself, so values with the same name in one package conflict.

**Source:** Protobuf Style Guide, Protobuf Best Practices

### Use dedicated request and response messages for each method [Recommended]

Name messages after the method. Do not share them between methods, and do not use a resource message directly as a request.

**Example:**

```proto
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);
}
```

**Source:** Protobuf Style Guide, Google AIP

### Prefer common types [Recommended]

Use `Timestamp` for time, `Duration` for durations, `FieldMask` for partial updates, and `Money` for amounts. Do not use custom representations such as `int64 timeout_millis`.

**Source:** Protobuf Best Practices

### Separate API messages from storage messages [Recommended]

Do not use RPC messages for long-term storage. The two change at different rates, and a conversion layer connects them.

**Source:** Protobuf Best Practices

### Paginate list methods from the start [Recommended]

Use `page_size`, `page_token`, and `next_page_token`. Do not use streaming responses in place of pagination. An empty `next_page_token` is the only signal of the end. Token rules are the same as for cursors in [rest.md](rest.md).

**Source:** Google AIP

## Error codes

### Use standard status codes [Required]

Return business failures through status. Do not put error fields in successful responses. Applications use only the status codes defined by gRPC.

| Scenario | Status code |
|---|---|
| The parameter itself is invalid, regardless of system state | `INVALID_ARGUMENT` |
| Beyond the currently valid range, such as reading past the end of a file | `OUT_OF_RANGE` |
| Resource does not exist | `NOT_FOUND` |
| The resource being created already exists | `ALREADY_EXISTS` |
| Caller cannot be identified | `UNAUTHENTICATED` |
| Caller has no permission | `PERMISSION_DENIED` |
| Quota or resources exhausted | `RESOURCE_EXHAUSTED` |
| System state is not satisfied; do not retry until it is fixed | `FAILED_PRECONDITION` |
| Concurrency conflict; retry from the read-modify-write | `ABORTED` |
| Temporarily unavailable; this call can be retried | `UNAVAILABLE` |
| Internal invariant broken | `INTERNAL` |

**Why:** Clients rely on status codes to decide whether to retry and at which layer.

**Source:** gRPC Docs

### Use structured messages for error details [Recommended]

When using Protobuf, carry `ErrorInfo` in the `details` of `google.rpc.Status`. `reason` is a stable `UPPER_SNAKE_CASE` identifier, and dynamic content goes in `metadata`. Clients do not parse `message`.

**Exceptions:** Details live in trailing metadata, which proxies and logging components cannot see, and language support varies. Put information these components need to observe in the status code.

**Source:** gRPC Docs, Google AIP

### Do not expose internal details in error messages [Required]

For unknown errors, return `INTERNAL` and a generic message, and record the full context in logs. Do not put stack traces in error details.

## Deadlines and retries

### Clients must set a Deadline [Required]

gRPC has no Deadline by default, so calls can wait forever. Estimate it from network latency and processing time.

**Source:** gRPC Docs

### Propagate Deadlines and cancellation downstream [Recommended]

When a server calls downstream, reuse the remaining time of the inbound call. Do not set a new, longer timeout. Some languages require propagation to be enabled explicitly.

**Source:** gRPC Docs

### Respond to cancellation on the server [Recommended]

When a Deadline expires, the framework cancels the call but does not interrupt business processing. Long-running work must check periodically whether the call has been canceled and stop any work it started.

**Source:** gRPC Docs

### Do not assume an operation failed after a timeout [Required]

On `DEADLINE_EXCEEDED`, a write may have already succeeded. Writes that need retries must have an idempotency guarantee, such as a `request_id` in the request.

**Source:** gRPC Docs, Google AIP

### Configure retry policies explicitly by status code [Recommended]

Automatically retry only idempotent unary calls, usually configuring only `UNAVAILABLE`, with a capped number of attempts and exponential backoff. Retry `ABORTED` at the transaction layer. Do not automatically retry `INVALID_ARGUMENT`, `DEADLINE_EXCEEDED`, or `INTERNAL`.

**Source:** gRPC Docs, Google AIP

## Streaming

### Use streams only when the benefit is clear [Recommended]

Use streaming only for long-lived, continuous data flows. Once a stream is established it cannot be load balanced again, and troubleshooting is harder. Use unary calls for ordinary request-response.

**Source:** gRPC Docs

### Handle interruption and backpressure in streaming calls [Required]

**Applies to:** production service

When a connection drops, in-progress streams fail and unsent data is lost. The protocol must let clients resume from the point of interruption. Writes follow the framework's flow control. Do not buffer outgoing messages without limit in the application layer.

**Source:** gRPC Docs

### Negotiate keepalive for long-lived connections with the service owner [Recommended]

Do not set the client keepalive interval far below 1 minute, and do not enable it when there are no calls.

**Source:** gRPC Docs

### Reuse Channels and Stubs [Recommended]

Hold Channels long-term. Do not create a new one for each call.

**Source:** gRPC Docs

## Interceptors

### Put cross-cutting logic in interceptors [Recommended]

Put metadata, logging, metrics, and server-side authentication and authorization checks in interceptors. Keep business rules in method implementations. For client authentication, prefer call credentials.

**Source:** gRPC Docs

### Define interceptor order [Recommended]

Interceptors close to the network see real network calls, and those close to the application see application behavior. For example, a logging interceptor must sit on the application side of a caching interceptor to record calls that hit the cache. Authentication must run before interceptors that depend on identity.

**Source:** gRPC Docs

### Do not manage connections with interceptors [Recommended]

Interceptors run per call. Set TCP, ports, and TLS in the Channel or server configuration.

**Source:** gRPC Docs
