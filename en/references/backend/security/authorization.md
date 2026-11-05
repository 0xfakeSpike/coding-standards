# Authorization

This document covers "what you can do" once identity is confirmed. For where identity comes from, see [Authentication](authentication.md). For recording authorization failures, see [Audit and abuse prevention](audit-and-abuse.md).

## Basic requirements

### Deny by default [Required]

Every endpoint requires authorization by default, and public endpoints are declared explicitly. Deny when no rule matches. Use explicit configuration, not framework default behavior.

**Why:** A forgotten configuration then results in denied access, not public exposure.

**Tooling:** In integration tests, enumerate all routes and assert that routes not declared public return 401 when not logged in.

**Source:** OWASP Authorization Cheat Sheet, OWASP API Security Top 10

### Check on a trusted server for every request [Required]

Check permissions on every request. Apply this through global middleware or interceptors, instead of relying on each handler to remember the call. Hiding a button on the client only improves the experience and is not access control. Static assets and export files need checks too.

**Source:** OWASP Authorization Cheat Sheet, OWASP ASVS

### Take user identity from the authentication context [Required]

Read the actor's identity, tenant, and role from the verified token or session. Do not trust `userId` or `role` in request parameters or the request body.

**Source:** OWASP Multi-Tenant Security Cheat Sheet

### Fail securely on authorization errors [Required]

Treat an error thrown during the check as a denial. Centralize failure handling, and do not expose internal details in error messages.

**Why:** Letting requests through on failure paths is a common cause of authorization bypass.

**Source:** OWASP Authorization Cheat Sheet, OWASP ASVS

### Apply permission changes quickly [Required]

**Applies to:** production service

Role and membership changes take effect immediately. When permissions are in a self-contained token and cannot take effect immediately, shorten the token lifetime, and requery current permissions for high-risk operations.

**Source:** OWASP ASVS

### Least privilege [Required]

**Applies to:** production service

Grant users, database accounts, cloud roles, and service accounts only the permissions they need.

**Source:** OWASP Authorization Cheat Sheet

## Choosing a model

### Choose a model that fits the shape of the authorization problem [Recommended]

| Model | Decides by | Use for |
|---|---|---|
| RBAC | Roles the user has | Coarse permissions such as feature entry points and admin menus |
| ABAC | Attributes of subject, resource, and environment | Rules that depend on data state, department, or time |
| ReBAC | Relationship between subject and resource | Owners, collaborators, sharing |

Roles alone struggle to express object-level rules, and the number of roles tends to balloon. When you need object-level or cross-organization rules, prefer ABAC or ReBAC.

**Source:** OWASP Authorization Cheat Sheet

### Define permission decisions centrally [Recommended]

Business code calls a single authorization module to ask "can this action be performed", for example `authorize(user, "order:refund", order)`. Do not scatter checks like `role == "admin"`.

**Source:** OWASP API Security Top 10

## Object-level and field-level permissions

### Check object-level permissions [Required]

Checking login is not enough. Also check whether the current user can operate on this record. Every function that accesses data by a client-supplied ID must check. This is the most common broken access control vulnerability (IDOR or BOLA).

Avoid:

```sql
SELECT id, amount FROM orders WHERE id = ?;
```

Prefer:

```sql
SELECT id, amount FROM orders WHERE id = ? AND user_id = ?;
```

Return 404 when nothing is found, without distinguishing "does not exist" from "no access".

**Why:** Otherwise any logged-in user can read other people's data by changing the ID.

**Tooling:** For each object endpoint, write a negative test that accesses it as another user.

**Source:** OWASP API Security Top 10, OWASP Authorization Cheat Sheet, OWASP ASVS

### Random IDs do not replace permission checks [Required]

Unpredictable IDs make enumeration harder, but IDs appear in logs, links, and shares. They are not access control.

**Source:** OWASP Authorization Cheat Sheet, OWASP Multi-Tenant Security Cheat Sheet

### Check function-level permissions [Required]

Check permissions separately on admin endpoints. Do not rely on obscure paths. Admin actions in regular controllers need checks too.

**Source:** OWASP API Security Top 10

### Check field-level permissions [Required]

Check per field on both reads and writes. Responses return only the fields the current caller is allowed to see, and writes accept only the fields the caller is allowed to change. For write rules, see [Input and injection](input-and-injection.md).

**Source:** OWASP API Security Top 10, OWASP ASVS

## Multi-tenant isolation

### Get tenant context from server-side verification [Required]

Determine the tenant at the start of the request, and bind it to the verified identity and current membership. A tenant ID in a request header or parameter is only a selector. You must verify that the caller belongs to that tenant.

**Source:** OWASP Multi-Tenant Security Cheat Sheet

### Enforce tenant conditions in the data access layer [Required]

Apply tenant conditions in one layer that all requests pass through, not by adding them query by query in business code. Shared tables can use row-level security as a second line of defense, in which case the database account on the request path must not be able to bypass the policy. Bind tenant context to the transaction, and clear it before returning the connection. Fail immediately when tenant context is missing. Do not fall back to a query without a tenant condition.

**Why:** With pooled connections, a session-level tenant setting can leak into the next request.

**Tooling:** For each tenant table, write a negative test for cross-tenant access.

**Source:** OWASP Multi-Tenant Security Cheat Sheet, OWASP ASVS

### Isolate caches, files, and async tasks too [Required]

| Location | Requirement |
|---|---|
| Cache | For values that vary by tenant, include the tenant in the key. Authorize before reading the cache |
| File storage | Isolate paths or buckets by tenant, and check ownership on access |
| Queues and tasks | Tenant context comes from an authenticated producer. Consumers rebuild the context and authorize |
| Idempotency keys and deduplication | Partition by tenant when results vary by tenant |
| Rate limits and quotas | Meter per tenant so one tenant cannot exhaust shared resources |

For cache key design, see [Cache and Redis](../data/cache-and-redis.md). For message design, see [Messaging](../async/messaging.md).

**Source:** OWASP Multi-Tenant Security Cheat Sheet

### Authorize cross-tenant operations explicitly [Required]

Cross-tenant paths such as operations and data repair use a separate identity and authorization, and write audit logs. Regular request paths contain no cross-tenant queries.

**Source:** OWASP Multi-Tenant Security Cheat Sheet

## Service-to-service authorization

### Authenticate and authorize internal calls too [Recommended]

Being on an internal network is not a basis for trust. Services authenticate each other, and the callee validates the caller's identity and token audience. Prefer mTLS or short-lived workload identity tokens over long-lived shared static keys.

**Source:** OWASP ASVS, OWASP Multi-Tenant Security Cheat Sheet

### Access with the initiator's permissions [Required]

**Applies to:** production service

When a service acts on a user's behalf, pass the user's identity and decide by the user's permissions. Do not read directly with the service account's high privileges.

**Why:** Otherwise the intermediate service becomes a high-privilege entry point others can borrow.

**Source:** OWASP ASVS

### Users must not be able to forge gateway-injected headers [Required]

For identity headers written by the gateway, such as `X-User-Id`, first strip externally supplied headers of the same name. The backend accepts only traffic from the gateway. Otherwise, do not trust these headers.

**Source:** OWASP ASVS

## Verification

### Authorization logic must have automated tests [Required]

**Applies to:** production service

Cover default deny, privilege escalation, cross-tenant access, and failure paths. Do not deploy when these tests fail.

**Source:** OWASP Authorization Cheat Sheet, OWASP API Security Top 10
