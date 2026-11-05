# Messaging

This document covers asynchronous communication over message queues, focusing on Kafka and RabbitMQ. For consistency between database writes and message sends (Outbox), see [consistency.md](../reliability/consistency.md). For job queues, see [background-jobs.md](background-jobs.md).

## Delivery semantics

### Design for at-least-once delivery [Required]

Once a message reaches the broker, design for at-least-once by default: not lost, but possibly duplicated. For the trade-offs between committing business data and the message reaching the broker, see Outbox in [consistency.md](../reliability/consistency.md). Do not rely on the exactly-once that middleware claims.

| Semantics | Meaning | Use for |
|---|---|---|
| At-most-once | May be lost, no duplicates | Monitoring samples that can be lost |
| At-least-once | Not lost, may be duplicated | Default choice |
| Exactly-once | Processed only once | Holds only within a specific scope |

**Why:** A producer that receives no acknowledgment can only resend, and a consumer that crashes after processing but before committing receives the message again. Kafka's exactly-once is limited to scenarios where reads and writes both stay inside Kafka. Writing to an external system requires that system's cooperation.

**Source:** Kafka Docs, RabbitMQ Docs

### Confirm the broker received the message [Required]

After sending, wait for broker acknowledgment, and resend unacknowledged messages. Do not send important messages in fire-and-forget mode.

| Middleware | Configuration |
|---|---|
| Kafka | `acks=all`, configure `min.insync.replicas`, disable unclean leader election |
| RabbitMQ | Enable publisher confirms, use durable queues and persistent messages |

**Why:** Resending creates duplicates, and consumer idempotency is the fallback for them. Without acknowledgment, a broker crash can lose messages not yet flushed to disk.

**Source:** Kafka Docs, RabbitMQ Docs

### Acknowledge after processing finishes [Required]

Consumers finish processing first, then commit the offset or send the ack. In RabbitMQ, do not use automatic acknowledgment mode for important messages.

**Why:** If you acknowledge before processing, a consumer crash loses the message. RabbitMQ automatic acknowledgment also has no cap on unacknowledged messages, so the consumer can be overwhelmed.

**Source:** Kafka Docs, RabbitMQ Docs

## Idempotent consumption

### Keep consumers idempotent [Required]

Processing the same message several times gives the same result as processing it once. Prefer making the operation itself idempotent, for example a conditional update or a write keyed by a business key. When that is not possible, deduplicate by message ID. For general idempotency approaches, see [consistency.md](../reliability/consistency.md).

**Example:**

Avoid: incrementing on every message received.

```sql
UPDATE accounts SET balance = balance + 100 WHERE id = ?;
```

Prefer: putting the deduplication record and the business write in the same transaction, and skipping the business write when the insert affects 0 rows.

```sql
INSERT INTO processed_messages (message_id, consumer) VALUES (?, ?)
ON CONFLICT (message_id, consumer) DO NOTHING;
UPDATE accounts SET balance = balance + 100 WHERE id = ?;
```

The example uses PostgreSQL syntax. MySQL uses `INSERT IGNORE`. RabbitMQ's `redelivered` flag only says the message may have been processed. It does not replace idempotent handling.

**Source:** RabbitMQ Docs

## Ordering

### Do not assume global ordering [Required]

Kafka guarantees order only within a single partition. RabbitMQ queues keep messages in publish order, but parallel processing by several consumers and requeueing both disrupt the observed order.

**Source:** Kafka Docs, RabbitMQ Docs

### Partition by business key when order matters [Recommended]

When messages for the same entity must stay in order, Kafka uses the entity ID as the message key so they go to the same partition. RabbitMQ uses a single active consumer. Both limit the parallelism of this data.

**Source:** Kafka Docs, RabbitMQ Docs

### Let consumers decide when order cannot be guaranteed [Recommended]

Messages carry a sequence number or business time. The consumer compares it with the last processed value and discards stale messages.

**Source:** Azure: Background Jobs

## Retry and dead letters

### Distinguish transient and permanent failures [Required]

Retry transient failures such as a downstream timeout. Do not retry permanent failures such as bad formats. Send them straight to the dead-letter queue. Missing referenced data may just be out-of-order delivery or replication lag, so treat it as transient first with a limited number of retries, then send it to the dead-letter queue once they run out.

**Why:** A permanent failure will never succeed no matter how many retries, and only wastes resources.

**Source:** Azure: Background Jobs

### Give failed messages a retry limit and a dead-letter queue [Required]

**Applies to:** production service

Cap retries and use backoff. Messages over the limit go to a dead-letter queue, with the failure reason recorded and an alert configured. Someone must handle dead letters. They cannot just pile up.

**Why:** Repeatedly failing messages block the queue or consume resources in an endless loop.

**Tooling:** Configure alerts on dead-letter count and the age of the oldest message. See [metrics-and-alerts.md](../observability/metrics-and-alerts.md).

**Source:** Azure: Competing Consumers, Azure: Background Jobs

### Make dead letters redeliverable [Recommended]

After the problem is fixed, dead-letter messages must be able to go back to the original queue. Keep the original content, failure reason, and retry count in the message. Replay relies on consumer idempotency.

## Message format

### Give messages uniform metadata [Recommended]

All messages use the same envelope, containing a unique ID, type, occurrence time, and trace ID. Business content goes in `data`.

**Example:**

```json
{
  "message_id": "01J9ZK3Q7X8Y",
  "type": "order.paid",
  "occurred_at": "2026-11-05T08:30:00Z",
  "trace_id": "a1b2c3",
  "data": {
    "order_id": "1001",
    "paid_at": "2026-11-05T08:30:00Z"
  }
}
```

### Name events for what already happened [Recommended]

Events use the past tense, in the format `entity.action`, for example `order.paid`. Commands use imperative meaning, for example `order.close`.

### Keep message bodies lean [Recommended]

Include only the fields consumers need. Put large objects in object storage and a reference in the message. Messages do not contain sensitive information such as passwords and tokens. Handle personal information according to [privacy.md](../security/privacy.md).

## Kafka essentials

### Plan partitions by consumption parallelism [Recommended]

Within a consumer group, a partition is handled by only one consumer at a time. Consumers beyond the partition count sit idle.

| Item | Practice |
|---|---|
| Message key | Use the entity ID when local ordering is needed; watch for data skew |
| Durability | Accept the latency of `acks=all` in exchange for not losing committed messages |
| Transactional consumption | When using Kafka transactions, set `isolation.level=read_committed` on consumers and disable auto-commit |
| Dead letters | Consumer groups have no built-in dead letters; the application forwards to a dead-letter topic |

**Source:** Kafka Docs

### Bound the processing time of a single batch [Required]

The interval between two polls by a Kafka consumer cannot exceed `max.poll.interval.ms`. Otherwise the consumer is kicked from the group and triggers a rebalance, and messages processed but not committed are repeated. For slow work, reduce the batch size or hand off to a job queue.

### Monitor consumer backlog [Recommended]

**Applies to:** production service

Monitor Kafka consumer lag and RabbitMQ queue length, and alert when the backlog keeps growing.

## RabbitMQ essentials

### Use quorum queues and limit prefetch [Recommended]

| Item | Practice |
|---|---|
| Queue type | Use replicated quorum queues for important data |
| Prefetch | Set a bounded `prefetch`; a few hundred is enough, and larger values bring no throughput gain |
| Delivery limit | Configure `delivery-limit`; messages over it are dropped or dead-lettered, so dead lettering must be configured too |
| Dead-letter reliability | Default dead-letter forwarding has no acknowledgment and can lose messages; quorum queues support at-least-once dead lettering |
| Acknowledgment timeout | Messages unacknowledged for 30 minutes (the default) are redelivered; adjust this for long tasks |
| Connections | Enable heartbeats to detect broken connections promptly |

**Source:** RabbitMQ Docs
