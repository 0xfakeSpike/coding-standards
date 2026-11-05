# REST API standards

This document covers the design of HTTP resource APIs. For contract management, see [contracts.md](contracts.md). For authentication and authorization, see [authentication.md](../security/authentication.md) and [authorization.md](../security/authorization.md). For server-side caching, see [cache-and-redis.md](../data/cache-and-redis.md).

## Resource modeling

### URLs identify resources, methods express actions [Recommended]

Paths use plural nouns in lowercase `kebab-case`, with no verbs and no trailing slash.

**Example:**

Avoid:

```text
POST /getOrderList
POST /order_items/delete
```

Prefer:

```text
GET    /orders
POST   /orders
GET    /orders/{id}
PATCH  /orders/{id}
DELETE /orders/{id}
GET    /users/{id}/orders
```

**Source:** Zalando RESTful API Guidelines

### Nest paths at most 2 levels [Recommended]

Nest only when the child resource's lifecycle depends on the parent. When nesting gets too deep, make the child a top-level resource.

### Model actions as resources first [Recommended]

First consider whether the action can be expressed as a state field or a sub-resource. When it truly cannot map to create, read, update, or delete, use `POST /orders/{id}/cancel`.

**Source:** Zalando RESTful API Guidelines

## Methods and status codes

### Follow HTTP method semantics [Required]

| Method | Purpose | Safe | Idempotent |
|---|---|---|---|
| GET | Read | Yes | Yes |
| POST | Create or perform an action | No | No |
| PUT | Replace entirely | No | Yes |
| PATCH | Partial update | No | Depends on the implementation |
| DELETE | Delete | No | Yes |

GET has no business side effects and no request body. When query conditions exceed the URL length, use POST instead and state this in the contract.

**Why:** Crawlers, prefetching, and caches can all trigger GET, and clients and proxies automatically resend idempotent requests.

**Source:** RFC 9110, Zalando RESTful API Guidelines

### Use an explicit patch format for PATCH [Recommended]

Choose only one per endpoint. Prefer `application/merge-patch+json`, and use `application/json-patch+json` when you need to operate on array elements.

**Source:** Zalando RESTful API Guidelines

### Use semantically correct status codes [Required]

Do not return 200 for all errors. Otherwise gateways, monitoring, and client retry logic cannot work.

| Status code | Scenario | Requirement |
|---|---|---|
| 200 / 204 | Success / success with no response body | |
| 201 | Created | Use `Location` to point to the new resource |
| 202 | Accepted, not yet complete | Provide an address for checking progress |
| 400 | Request cannot be parsed | |
| 401 | Not authenticated | Must include `WWW-Authenticate` |
| 403 | Authenticated but no permission | |
| 404 | Resource does not exist, or its existence should not be revealed | |
| 405 | Resource does not support the method | Must include `Allow` |
| 409 | Conflict with current state | Response explains the conflict, with no field errors |
| 412 | Precondition of a conditional request not met | |
| 415 | Media type not supported | |
| 422 | Schema, field, or business validation failed | Response includes `errors` |
| 429 | Rate limit exceeded | Include `Retry-After` |
| 500 / 502 / 503 / 504 | Server or upstream error | 503 may include `Retry-After` |

**Source:** RFC 9110, Zalando RESTful API Guidelines

## Error format

### Use Problem Details for error responses [Recommended]

Error responses in new projects use RFC 9457 with the media type `application/problem+json`. `type` is an absolute URI, `status` must equal the actual status code, and validation details and the request identifier go in extension fields.

**Example:**

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/order-already-paid",
  "title": "Order already paid",
  "status": 409,
  "detail": "The order has been paid and cannot be canceled",
  "requestId": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "requestId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    { "pointer": "#/couponId", "detail": "Coupon has expired" }
  ]
}
```

**Exceptions:** When a project has already standardized on another error format, follow the next rule.

**Source:** RFC 9457, Zalando RESTful API Guidelines

### Keep an existing standardized response wrapper [Recommended]

When a project already uses a wrapper such as `{ code, message, data }` throughout, keep it. Do not mix two error formats in one project. Field errors go in `errors`, with the same structure as Problem Details. HTTP status codes must still reflect the real result. Do not use 200 plus a business code to signal failure.

**Example:**

Avoid:

```http
HTTP/1.1 200 OK

{ "code": 40901, "message": "Order already paid", "data": null }
```

Prefer:

```http
HTTP/1.1 409 Conflict

{ "code": "ORDER_ALREADY_PAID", "message": "The order has been paid and cannot be canceled", "data": null }
```

### Give errors a stable machine-readable identifier [Recommended]

In Problem Details this is `type`. In a response wrapper it is `code`. Clients decide based on the identifier and do not parse `detail` or `message`.

**Source:** RFC 9457

### Do not expose internal details in error responses [Required]

Do not return stack traces, SQL, or internal service addresses. For general principles, see [error-handling.md](../../general/error-handling.md).

**Source:** RFC 9457, Zalando RESTful API Guidelines

### Return a request identifier in errors [Recommended]

Return `requestId` in a response header or the error body, set to the `traceId` generated at the entry point, so logs can be located when users report problems.

## Field conventions

### Represent fields consistently across the project [Recommended]

| Content | Requirement |
|---|---|
| Naming | Choose `camelCase` or `snake_case` for the whole project. When it differs from language naming, map it with serialization configuration |
| Time | Fields representing an instant use RFC 3339 UTC with `Z`, such as `2026-11-05T08:30:00Z`; local dates use `YYYY-MM-DD`; future events use local time plus an IANA time zone |
| Amounts | Do not use floats. Choose either integers in the smallest currency unit or decimal strings for the whole project, and always include the ISO 4217 currency |
| 64-bit integer IDs | Always declare as string in the contract, regardless of the current value size. Otherwise JavaScript loses precision when parsing |
| Enums | Use strings, such as `"paid"`, not `3` |
| Empty lists | Return `[]`, not `null` |

## Pagination, filtering, and sorting

### Paginate list endpoints by default and set a limit [Required]

**Applies to:** production service

For example, default 20, maximum 100. State the limit in the contract. When a request exceeds the limit, apply the limit and return the effective `limit` in the response. Return 422 for negative values.

**Why:** Unpaginated lists drag down the service as data grows.

**Source:** Google AIP, Zalando RESTful API Guidelines

### Use cursor pagination for large or frequently changing lists [Recommended]

`OFFSET` performs poorly on deep pages and can repeat or skip items when data changes. Admin views with small data volumes can use page-number pagination.

**Example:**

```text
GET /orders?status=paid&sort=-createdAt&limit=20&cursor=eyJpZCI6MTAwfQ
```

**Source:** Zalando RESTful API Guidelines

### Keep cursors opaque to clients [Recommended]

Clients neither parse nor construct cursors. Return 422 when filter conditions change while paging. A cursor represents only a position, so authorize each request as usual. The last page returns no next-page cursor.

**Source:** Google AIP, Zalando RESTful API Guidelines

### Do not return totals by default [Recommended]

Counting is expensive for complex queries. When truly needed, provide it separately, and note estimated values in the contract.

**Source:** Zalando RESTful API Guidelines

### Give filter and sort parameters clear semantics [Recommended]

Use query parameters for simple filters. Multiple conditions combine with AND. Use `sort` for sorting, with a `-` prefix for descending order. State the comparison method and value range for each parameter.

**Source:** Zalando RESTful API Guidelines

## Batch operations and long-running tasks

### Declare atomicity and limits for batch endpoints [Recommended]

By default, all items succeed or all fail. When partial success is allowed, return 207, with an identifier and status for each item in the response. Set a limit on batch size.

**Source:** Google AIP, Zalando RESTful API Guidelines

### Return a task resource for long-running tasks [Recommended]

When a request may exceed the gateway timeout, return 202 and `Location`. The task resource contains the status and the result or error. While it is incomplete, use `Retry-After` to suggest a polling interval. After completion, retain it for a publicly stated period. When the task fails, the query endpoint still returns 200, and the failure goes in the task's `error` field.

**Example:**

```http
HTTP/1.1 202 Accepted
Location: /report-jobs/8f3a
Retry-After: 5
```

**Source:** Microsoft REST API Guidelines, Google AIP, Zalando RESTful API Guidelines

## Idempotency and concurrency

### Support idempotency for critical writes [Required]

**Applies to:** high risk

For operations such as payments and placing orders, the client sends `Idempotency-Key`. The server stores the key, a request digest, and the response, and returns the same result for the same key within the validity period. Write the key record and the local state in the same transaction. When an external service must be called, follow the order in [transactions.md](../data/transactions.md). When a natural business unique key exists, implement this with a unique constraint.

**Why:** After a timeout, the client cannot tell whether the request succeeded and can only retry.

**Source:** Zalando RESTful API Guidelines, Google AIP

### Define responses for idempotency key edge cases [Required]

**Applies to:** high risk

| Case | Response |
|---|---|
| Key is required but missing | 400 |
| Same key, different request digest | 422 |
| A request with the same key is still running | 409 |

Replay only completed results. Failures rolled back by the transaction leave no record, so the client can retry with the same key.

### Use optimistic locking for concurrent updates [Recommended]

Use `ETag` with `If-Match`, or a `version` field. Return 412 when a conditional request fails, and 409 on a `version` conflict.

**Source:** RFC 9110, Zalando RESTful API Guidelines

## Caching and rate limiting

For rate limiting on public APIs, see [resilience.md](../reliability/resilience.md).

### Do not cache by default, declare cacheable endpoints explicitly [Recommended]

Dynamic and user-specific responses use `Cache-Control: no-store` by default, and responses containing personal data must use it. See [privacy.md](../security/privacy.md). Cacheable endpoints declare `Cache-Control`, `ETag`, and `Vary`, and support `If-None-Match` returning 304.

**Source:** RFC 9110, Zalando RESTful API Guidelines
