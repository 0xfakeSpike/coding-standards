# Realtime communication

This document covers approaches for pushing from server to client. For the authentication model, see [authentication.md](../security/authentication.md). For cross-site risks, see [web-risks.md](../security/web-risks.md). For general backpressure strategies, see [resilience.md](../reliability/resilience.md).

## Choosing an approach

### Choose the simplest approach that meets the need [Recommended]

| Approach | Use when | Notes |
|---|---|---|
| Polling | Updates are infrequent and a few seconds of delay is acceptable | Simplest, holds no long-lived connection |
| SSE | Only server push is needed, such as notifications, progress, streaming output | HTTP-based, built-in reconnection, UTF-8 text only |
| WebSocket | Two-way, high-frequency communication, such as collaboration or chat | You implement authorization, heartbeats, and reconnection yourself |

When the client only sends messages occasionally, receive with SSE and send with ordinary HTTP requests.

### Use HTTP/2 for SSE [Recommended]

Under HTTP/1.1, browsers allow only about 6 connections per domain, shared across tabs. Opening several tabs exhausts them.

**Source:** MDN

### Configure proxies along the path [Recommended]

Reverse proxies and load balancers must support WebSocket upgrades, and the read timeout for long-lived connections must be longer than the heartbeat interval. SSE responses use `text/event-stream` and `Cache-Control: no-cache`, with proxy buffering turned off.

**Example:**

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
X-Accel-Buffering: no
```

**Why:** Proxy buffering makes events pile up, and proxy timeouts periodically cut idle connections.

**Source:** MDN, OWASP WebSocket Security Cheat Sheet

## Authorization

### Use wss and https in production [Required]

**Applies to:** production service

Do not use unencrypted `ws://`.

**Source:** OWASP WebSocket Security Cheat Sheet

### Validate Origin during the handshake [Required]

Compare the full Origin against an explicit allowlist. Do not use wildcards or substring matching. Return 403 when the check fails. With cookie authentication, combine this with `SameSite` and CSRF tokens.

**Why:** Browsers send cookies automatically during the handshake, so a malicious site can open a connection using the user's identity.

**Boundary:** Origin only protects browser scenarios. Non-browser clients can forge it, so it does not replace authentication.

**Source:** OWASP WebSocket Security Cheat Sheet, MDN

### Do not put tokens in URLs [Required]

URLs end up in access logs. For WebSocket, prefer sending the token in the first message after the connection opens. Before authentication completes, accept only authentication messages and send no protected data. Close the connection on failure or timeout. Message logs must not record tokens either.

Native `EventSource` can only carry cookies through `withCredentials`. When you need an `Authorization` header, switch to a fetch-based client.

**Example:**

Avoid:

```text
wss://api.example.com/socket?token=eyJhbGciOi
```

Prefer:

```json
{ "type": "auth", "token": "eyJhbGciOi" }
```

**Source:** OWASP WebSocket Security Cheat Sheet

### Change connection permissions with the session [Required]

**Applies to:** high risk

Close the corresponding connections when a session expires or the user logs out, and revalidate long-lived connections periodically. This requires maintaining a mapping from sessions to connections.

**Source:** OWASP WebSocket Security Cheat Sheet

### Authorize and validate each message separately [Required]

An open connection does not mean every operation is allowed. Check permissions for each action and subscription, validate message structure as untrusted input, and limit size and frequency.

**Source:** OWASP WebSocket Security Cheat Sheet

### Limit the number of connections [Required]

**Applies to:** production service

Limit concurrent connections per user and per IP, with stricter limits for unauthenticated connections, and set an authentication timeout.

**Source:** OWASP WebSocket Security Cheat Sheet

## Heartbeats

### Long-lived connections must have heartbeats [Required]

**Applies to:** production service

The server sends heartbeats periodically. When responses time out repeatedly, close the connection and release its resources. The interval must be shorter than the shortest idle timeout along the path.

| Protocol | Method |
|---|---|
| WebSocket, server probes | Ping frame, peer replies with Pong |
| WebSocket, browser probes | Application-level heartbeat messages, because the browser API cannot send Ping |
| SSE | Periodically send comment lines starting with a colon |

**Example:**

```text
: ping

event: order.updated
id: 1042
data: {"orderId":"1001","status":"paid"}

```

**Why:** When the network drops, the peer may receive no notification at all, and half-open connections hold resources indefinitely.

**Source:** MDN

## Reconnection

### Use backoff and jitter for client reconnection [Required]

**Applies to:** production service

Use exponential backoff with random jitter, and set a cap. Do not reconnect after unrecoverable closes such as authorization failures. SSE can set the reconnection wait time with the `retry` field.

**Why:** When a service restarts, all clients reconnecting at once create a traffic spike.

### Backfill missed messages after reconnection [Recommended]

SSE events carry an `id`. On reconnection, the browser reports the last received event through the `Last-Event-ID` request header. For WebSocket, the client sends the last sequence number after reconnecting. The server retains messages for a period, and notifies the client to refetch all data when the gap exceeds the retention window.

**Exceptions:** Data where only the latest value matters, such as live market quotes, need not be backfilled.

### Do not make push the only data source [Recommended]

Use push for timely notification, and make the full state readable through an HTTP API. After reconnection, the API result is authoritative.

### Notify clients before going offline [Recommended]

Before a service stops, send a close frame and have clients reconnect to other instances in batches. For graceful shutdown, see [operations.md](../reliability/operations.md).

## Message ordering

### Give messages sequence numbers [Recommended]

Messages are ordered within a single connection, but there is no global order after reconnection or across instances. Assign increasing sequence numbers per business entity or channel. Clients use them to detect gaps and out-of-order messages. Do not rely on arrival order or client time.

**Example:**

```json
{ "type": "order.updated", "channel": "order:1001", "seq": 42, "data": { "status": "paid" } }
```

### Keep message handling idempotent [Recommended]

Backfilling produces duplicates, so clients deduplicate by message ID or sequence number. When delivery acknowledgment is needed, implement it in the application-layer protocol.

### Use a uniform message envelope [Recommended]

Define fields such as `type`, `id`, and `data`, and use the same structure for errors. Put types and fields in the contract.

### Control send buffers on the server [Required]

**Applies to:** production service

The browser `WebSocket` API does not support backpressure, and messages arriving too fast fill up memory. Set a send queue limit for each connection on the server, and drop, merge, or disconnect slow clients according to policy.

**Source:** MDN

### Broadcast across instances through message middleware [Recommended]

Connections are spread across instances, which forward messages to each other through publish-subscribe. For delivery semantics, see [messaging.md](../async/messaging.md).
