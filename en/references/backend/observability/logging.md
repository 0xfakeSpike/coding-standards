# Logging

This document sets the format, levels, fields, and retention of backend logs. For audit logs, see [Audit and abuse prevention](../security/audit-and-abuse.md). For overall requirements on personal information, see [Privacy](../security/privacy.md).

## Structured logging

Service logs go to standard output. See [Containers](../../general/containers.md).

### Use structured logging [Recommended]

**Applies to:** production service

Output logs in a structured format such as JSON, and record them with a logging library, not standard output print functions. Keep the message template fixed and put variables in fields.

**Why:** Logs built by text concatenation cannot be searched or aggregated by field, and they are easier to inject into.

**Example:**

```json
{
  "timestamp": "2026-11-05T08:30:12.345Z",
  "level": "ERROR",
  "service": "payment",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "message": "Failed to process payment callback",
  "orderId": "1001",
  "error": "gateway timeout"
}
```

**Source:** OWASP Logging Cheat Sheet

### Do not let logging failures affect the business [Required]

**Applies to:** production service

When the logging system is unavailable, business requests keep being processed. They must not crash or block on writing logs.

**Source:** OWASP Logging Cheat Sheet

## Levels

Log each error only once. See [Error handling principles](../../general/error-handling.md).

### Choose the level by how it gets handled [Recommended]

| Level | Use | Example |
|---|---|---|
| ERROR | Failures that need human attention | Database unavailable, code defect |
| WARN | Abnormal but recovers automatically | Retry succeeded, graceful degradation in effect |
| INFO | Key business events | Order created, service started |
| DEBUG | Troubleshooting details, off by default in production | Intermediate calculation results |

Log business errors as INFO or WARN and system errors as ERROR. For the classification, see [Layered architecture](../architecture/layering.md).

**Why:** If user input errors are logged as ERROR, real failures get buried.

### Do not write heavy INFO on hot paths [Recommended]

Do not log each item in loops or on paths that run many times per second. When you need to know the frequency, use metrics. See [Metrics and alerts](metrics-and-alerts.md).

### Allow level changes at runtime [Recommended]

During troubleshooting, DEBUG can be turned on temporarily without a redeploy, and it reverts automatically when time is up.

## Field conventions

### Include base fields in every log entry [Recommended]

| Field | Meaning | Requirement |
|---|---|---|
| `timestamp` | Event time | ISO 8601, UTC |
| `level` | Level | Uniform values across the project |
| `service` | Service name | Same as the service name in distributed tracing |
| `traceId` | Trace identifier | Written when trace context exists |
| `message` | Event description | Fixed text |

Optional fields include `spanId`, `version`, `host`, `userId`, `error`, and `durationMs`.

**Source:** OWASP Logging Cheat Sheet, OpenTelemetry: Logs Specification

### Answer when, where, who, and what [Recommended]

Key events must state when they happened, where, who initiated them, and the outcome. When the system recording time differs from the event time, record both. Keep node clocks in sync, or the order of cross-service logs cannot be trusted.

**Source:** OWASP Logging Cheat Sheet

### Use field names consistently everywhere [Recommended]

The same meaning uses the same field name and type in all services. For example, always use `orderId`, and do not mix in `order_id` or `oid`.

**Why:** With inconsistent field names, you cannot search across services.

**Source:** OWASP Logging Cheat Sheet

### Correlate logs with traces [Recommended]

Write `traceId` when trace context exists, so you can jump from logs to traces. When a tracing library is integrated, let it inject the value automatically rather than passing it by hand. For details, see [Distributed tracing](tracing.md).

**Source:** OpenTelemetry: Logs Specification

### Include context in error logs [Recommended]

Record the failed operation, the object, and the reason, and keep the stack trace. A log that only says "operation failed" cannot be investigated.

## Redaction

### Content that must not be logged [Required]

Do not write the following to logs:

- Passwords, tokens, API keys, encryption keys, and connection strings containing credentials
- Full bank card numbers and ID numbers
- Data collected without a legal basis

**Why:** Logs have wider access than business databases and longer retention, and they are hard to fully purge after a leak.

**Tooling:** Configure field filters in the logging library, and assert in tests that sensitive fields do not appear.

**Source:** OWASP Logging Cheat Sheet

### Process personal and sensitive data before logging [Required]

| Data | Handling |
|---|---|
| Phone numbers, email | Mask, for example `138****0000` |
| Session identifiers | Hash, which can still correlate the same session |
| User identifiers | Use internal IDs when they are enough for troubleshooting, not names |
| Internal addresses and file paths | Log with care and do not expose externally |

**Example:**

Avoid:

```json
{ "message": "Login failed", "phone": "13800000000", "password": "abc123" }
```

Prefer:

```json
{ "message": "Login failed", "phone": "138****0000", "reason": "invalid_credentials" }
```

**Source:** OWASP Logging Cheat Sheet

### Do not log full requests and responses [Required]

By default, do not log request bodies, response bodies, or full request headers. When truly needed, specify fields with an allowlist.

**Why:** Denylists always miss something, and headers and bodies are where sensitive data is most concentrated.

### Prevent log injection [Required]

Before writing user input to logs, handle newlines and delimiters, and limit length. Structured output and fixed message templates stop most problems.

**Source:** OWASP Logging Cheat Sheet

## Sampling

### Do not sample errors and security events [Recommended]

**Applies to:** production service

Keep all ERROR, WARN, and security-related events. Sample only high-frequency, repetitive INFO and DEBUG.

**Why:** Sampled-out errors cannot be recovered, and the value of events such as authentication failures lies in their frequency.

### Keep traces intact when sampling, and cap total volume [Recommended]

Decide INFO and DEBUG sampling by `traceId`, so that these logs for one request are either all kept or all dropped. ERROR, WARN, and security events bypass sampling. Set limits on both single log entries and per-instance output rate, so a log flood during a failure cannot exhaust disk or the collection pipeline. When a limit is reached, drop lower-level logs first and record the number dropped.

**Source:** OWASP Logging Cheat Sheet

## Retention policy

### Set retention by legal, contractual, and troubleshooting needs [Required]

**Applies to:** high risk

Set a retention period for each log type, and delete on expiry. Do not destroy logs early or keep them indefinitely. Security and audit logs follow regulatory and contractual requirements. Application logs follow troubleshooting needs.

**Source:** OWASP Logging Cheat Sheet

### Restrict log access [Required]

**Applies to:** high risk

Grant access to the logging platform on least privilege, and do not put logs anywhere web-accessible.

**Source:** OWASP Logging Cheat Sheet
