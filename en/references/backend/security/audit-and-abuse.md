# Audit and abuse prevention

This document covers security auditing, rate limiting, anti-automation, and account risk control. For regular operational logging rules, see [Logging](../observability/logging.md). For alerts, see [Metrics and alerts](../observability/metrics-and-alerts.md). For dependency vulnerability scanning, see [Dependency management](../../general/dependencies.md).

## Audit logs

**Applies to:** high risk (all rules in this section)

For rules that prevent log injection, see [Logging](../observability/logging.md).

Audit logs also do not record sensitive content. See [Logging](../observability/logging.md) for the rules. When you need to correlate a session, record its hash.

### Record key operations [Required]

| Category | Events |
|---|---|
| Authentication | Login success and failure, logout, MFA changes, password changes and resets |
| Authorization | Authorization failures, permission and role changes, administrator actions |
| Data | Sensitive data access, exports, bulk deletes |
| Money | Payments, refunds, withdrawals |
| System | Configuration changes, key rotation, break-glass account use |
| Protection | Validation failures, deserialization failures, bypassed business flows, exceeded limits |

**Why:** Without these records, you cannot determine the impact of an incident.

**Source:** OWASP Logging Cheat Sheet, OWASP ASVS

### State who, when, where, and what in each record [Required]

Include at least the actor, time, source, action, target, and result. Use UTC with `Z` for time, and keep clocks synchronized across nodes. Multi-tenant systems record the verified tenant.

```json
{
  "time": "2026-11-05T08:30:12Z",
  "actor": "user:1024",
  "tenant": "t-77",
  "sourceIp": "203.0.113.7",
  "action": "order.refund",
  "target": "order:5531",
  "result": "denied",
  "requestId": "9f2c1e"
}
```

**Source:** OWASP ASVS, OWASP Logging Cheat Sheet, OWASP Multi-Tenant Security Cheat Sheet

### Separate audit logs from operational logs [Recommended]

The two differ in purpose, retention period, and who can access them. Store them separately.

**Source:** OWASP Logging Cheat Sheet, OWASP ASVS

### Application accounts must not be able to modify audit logs [Required]

Allow appends only, for example with a write-only database account or an append-only table. Deletion on expiry is done by the retention policy. See [Privacy](privacy.md). Restrict read access.

**Source:** OWASP Top 10, OWASP Logging Cheat Sheet, OWASP ASVS

### Alert on suspicious behavior [Recommended]

Recording without alerting is the same as not detecting. Configure alerts for login failure spikes, large numbers of authorization failures, and abnormal exports, and keep false positives under control.

**Source:** OWASP Top 10

## Rate limiting

**Applies to:** production service (all rules in this section)

### Rate-limit expensive endpoints separately [Required]

For endpoints such as login, code verification, SMS, email, password recovery, export, and search, set stricter thresholds based on the business, not just the global limit. Also limit the per-user frequency of the same operation.

| Endpoint | Limit by |
|---|---|
| Login | Account, IP, device |
| Send SMS | Phone number, account, IP, plus a global cap |
| Verify code | Attempts per code |
| Export | Account, tenant, concurrency |

**Source:** OWASP API Security Top 10

### Keep security-related rate limit counters in shared storage [Required]

For endpoints such as login, SMS, and export, where threshold precision affects security or cost, keep counters in shared storage such as Redis when deploying multiple instances. Otherwise, the actual threshold grows with the instance count. Return 429 with `Retry-After` when the limit is exceeded.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

### Get the client IP correctly [Required]

Trust only forwarded headers written by known proxies, and overwrite externally supplied values at the gateway. Otherwise, an attacker can bypass IP rate limiting by changing `X-Forwarded-For` on each request.

**Source:** OWASP ASVS

### Limit resource consumption [Required]

| Resource | Limits |
|---|---|
| Requests | Request body size, string length, array element count |
| Queries | Page size cap, query complexity and depth |
| Connections | Read timeout, absolute timeout, protection against slow attacks |
| Processes | Memory, CPU, file descriptors |
| Third parties | Spending cap or billing alerts |

Validate the page size cap on the server, and do not accept arbitrary client values. Multi-tenant systems meter per tenant. For timeouts and circuit breakers, see [Resilience](../reliability/resilience.md).

**Why:** A single request can exhaust resources, and SMS and cloud services also incur direct costs.

**Source:** OWASP API Security Top 10, OWASP Denial of Service Cheat Sheet, OWASP Multi-Tenant Security Cheat Sheet

## Anti-automation

**Applies to:** production service (all rules in this section)

### Identify sensitive business flows [Recommended]

List the flows that hurt the business when overused, such as registration, claiming coupons, flash sales, posting, and sending SMS. Every request in this kind of abuse is legitimate, so endpoint validation cannot detect it.

**Source:** OWASP API Security Top 10

### Enforce business limits on the server [Required]

Define and enforce both per-user and global limits. For limited-quantity resources, use transactions or locks to prevent overselling, and prevent concurrent duplicate claims of the same benefit.

**Source:** OWASP ASVS

### Slow down automation in layers [Recommended]

Combine by risk: device fingerprinting, human verification, non-human behavior analysis (for example, adding to cart and placing an order within one second), and IP intelligence on proxies and Tor exits. Credential stuffing tools rotate proxies, so do not base blocking decisions on a single threshold, and take legitimate users on shared exits into account. These measures only raise the cost and are not the sole line of defense. B2B and open APIs need the same protection.

**Source:** OWASP API Security Top 10, OWASP Credential Stuffing Prevention Cheat Sheet

## Account risk control

**Applies to:** high risk (all rules in this section)

### MFA is the main defense against credential stuffing [Required]

Require MFA for high-risk logins, such as a new device, an unusual location, or a known leaked password. Check against leaked password databases when a password is set and at login. For authentication rules, see [Authentication](authentication.md).

**Source:** OWASP Credential Stuffing Prevention Cheat Sheet, OWASP ASVS

### Bind failure counts to accounts [Required]

Count failures per account, not just per IP. The consecutive failure limit is at most 100, and the business can set it lower. Use increasing delays or temporary lockout. During lockout, keep a self-service recovery path for the user, so attackers cannot lock other people's accounts through repeated failures.

**Why:** When counting only per IP, proxy pools bypass the limit easily.

**Source:** OWASP Authentication Cheat Sheet, NIST SP 800-63B

### Reverify before high-risk operations [Required]

Before changing account bindings, withdrawing funds, or exporting all data, require reauthentication or additional verification.

**Source:** OWASP ASVS
