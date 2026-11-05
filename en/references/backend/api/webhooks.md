# Webhooks

This document covers sending and receiving webhooks. For key management, see [crypto-and-secrets.md](../security/crypto-and-secrets.md). For general SSRF protection, see [web-risks.md](../security/web-risks.md).

## Signing

Compare signatures with a constant-time comparison. See [crypto-and-secrets.md](../security/crypto-and-secrets.md).

### Sign every delivery [Required]

The signed content contains the message ID, timestamp, and request body, joined with `.`, and is passed through request headers.

| Request header | Content |
|---|---|
| `webhook-id` | Unique event identifier, unchanged across retries |
| `webhook-timestamp` | Unix timestamp in seconds for this attempt |
| `webhook-signature` | Signature list, space-separated |

**Example:**

```text
Signed content: msg_2KWPBgLl.1674087231.{"type":"invoice.paid"}
```

```http
POST /webhooks/billing HTTP/1.1
Content-Type: application/json
webhook-id: msg_2KWPBgLl
webhook-timestamp: 1674087231
webhook-signature: v1,K5oZfzN95Z9UVu1EsfQmfVNQhnkZ2pj9o9NDN/H/pI4=

{"type":"invoice.paid"}
```

**Why:** When only the body is signed, the timestamp and ID can be tampered with, and replay protection stops working.

**Source:** Standard Webhooks

### Choose a suitable signature algorithm [Recommended]

When you cannot control the security of both the sending and receiving sides, prefer asymmetric signatures, so the receiver holds only the public key.

| Type | Algorithm | Identifier | Key format |
|---|---|---|---|
| Symmetric | HMAC-SHA256 | `v1` | 24 to 64 random bytes, base64-encoded, prefix `whsec_` |
| Asymmetric | ed25519 | `v1a` | Prefix `whsk_` (private key), `whpk_` (public key) |

**Source:** Standard Webhooks

### Use a separate key for each endpoint [Required]

Isolate symmetric keys per endpoint. Isolate asymmetric keys at least per customer.

**Source:** Standard Webhooks

### Verify signatures against the raw request body [Required]

Do not parse JSON before verification. The bytes the sender signs must be exactly the bytes actually sent.

**Example:**

Avoid: parsing the JSON, re-serializing it, then computing the signature.

Prefer: reading the raw bytes, and parsing only after verification passes.

**Why:** Re-serializing can change whitespace and field order, which makes verification fail.

**Source:** Standard Webhooks

### Trust only preconfigured public keys [Required]

Do not use a public key carried in the request.

**Source:** Standard Webhooks

### Support key rotation without interruption [Recommended]

During rotation, sign once with each of the new and old keys and put both in the same request header. The receiver verifies them one by one, and any single pass is enough.

**Source:** Standard Webhooks

## Replay protection

### Validate timestamp tolerance [Required]

Reject when the difference between `webhook-timestamp` and the current time exceeds the tolerance, such as 5 minutes. The timestamp can be trusted only after signature verification passes. Senders update the timestamp and re-sign on every retry.

**Source:** Standard Webhooks

### Do not accept user control of message IDs and timestamps [Required]

At minimum, forbid `.` in them. Otherwise an attacker can construct identical signed content.

**Source:** Standard Webhooks

## Delivery and retries

Senders must set a timeout, 15 to 30 seconds recommended. For the rules, see [resilience.md](../reliability/resilience.md).

### Determine the result by status code [Recommended]

| Response | Sender action |
|---|---|
| 2xx | Success |
| 3xx | Failure, do not follow redirects |
| 410 | Disable the endpoint and stop sending |
| 429, 502, 504 | Failure, and reduce the sending rate |
| Other status codes, timeout, dropped connection | Failure, retry on schedule |

When the response includes `retry-after`, do not retry earlier than that time.

**Source:** Standard Webhooks

### Use exponential backoff and jitter for retries [Required]

**Applies to:** production service

The retry schedule spans several days and adds random jitter.

**Example:**

| Retry interval | Time since first send |
|---|---|
| Immediately | 0 |
| 5 seconds | 5 seconds |
| 5 minutes | About 5 minutes |
| 30 minutes | About 35 minutes |
| 2 hours | About 2.5 hours |
| 5 hours | About 7.5 hours |
| 10 hours | About 17.5 hours |
| 14 hours | About 31.5 hours |

**Source:** Standard Webhooks

### Decouple delivery from business writes [Required]

**Applies to:** production service

Persist the event first, then send it from a background job. Do not call the other party's address inside a business transaction. For reliable writes, see [consistency.md](../reliability/consistency.md).

### Receivers acknowledge first, then process [Recommended]

After verifying the signature and persisting, return 2xx immediately. Move slow processing to a background job.

**Why:** Synchronous processing easily times out and triggers duplicate deliveries.

### Notify and disable on long-term failure [Recommended]

When an endpoint keeps failing, stop delivery and notify the other party.

**Source:** Standard Webhooks

## Idempotency

### Receivers deduplicate by event ID [Required]

Use `webhook-id` as the idempotency key. Record processed events with a unique constraint, in the same transaction as the business write. The retention period for deduplication records must cover the sender's full retry cycle.

**Example:**

```sql
INSERT INTO webhook_events (event_id, received_at)
VALUES ('msg_2KWPBgLl', now())
ON CONFLICT (event_id) DO NOTHING;
```

**Why:** Network problems, retries, and malicious replay all cause duplicate deliveries.

**Source:** Standard Webhooks

### Do not rely on delivery order [Required]

Retries scramble order. Receivers determine what is newer by event time or resource version, or look up the latest state after receiving a notification.

## Message content

### Use a uniform event structure [Recommended]

Include `type`, `timestamp`, and `data`. `timestamp` is when the event occurred, not when it was delivered. Types use dot-separated hierarchical names. The structure of a given type is fixed and goes in the contract.

**Example:**

```json
{
  "type": "invoice.paid",
  "timestamp": "2026-11-05T08:30:00Z",
  "data": { "id": "inv_1001" }
}
```

**Source:** Standard Webhooks

### Control message body size [Recommended]

Keep it under 20 KB. For large data, send a link, or send only an identifier and let the receiver look it up. Sending only an identifier also reduces outbound sensitive data, and the receiver's lookup still goes through normal authorization.

**Source:** Standard Webhooks

## Sender security

Callback URLs are provided by users. When sending, protect against SSRF according to [web-risks.md](../security/web-risks.md), and run the sending process in a separate subnet that cannot reach internal services.

### Use HTTPS for callback URLs [Recommended]

Signatures guarantee only integrity. They do not encrypt the content.

**Source:** Standard Webhooks
