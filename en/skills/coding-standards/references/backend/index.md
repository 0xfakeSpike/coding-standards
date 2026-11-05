# Backend layer index

**Trade-off:** Judge whether every technique in this layer, including retries, fallbacks, compensation, Outbox, and circuit breakers, is worth it by [Do not add compensation for rare failures](reliability/resilience.md).

## APIs

| File | Read when |
|---|---|
| [api/rest.md](api/rest.md) | Designing HTTP APIs, error formats, pagination, idempotency |
| [api/contracts.md](api/contracts.md) | OpenAPI, schemas, code generation, contract tests |
| [api/grpc.md](api/grpc.md) | gRPC services and Proto |
| [api/graphql.md](api/graphql.md) | GraphQL schemas and query control |
| [api/realtime.md](api/realtime.md) | WebSocket and SSE |
| [api/webhooks.md](api/webhooks.md) | Sending or receiving webhooks |

## Architecture

| File | Read when |
|---|---|
| [architecture/layering.md](architecture/layering.md) | Layering, DTOs, where business logic goes |
| [architecture/domain-modeling.md](architecture/domain-modeling.md) | Domain modeling, whether to use DDD |
| [architecture/service-boundaries.md](architecture/service-boundaries.md) | Module and service boundaries, splitting services |
| [architecture/configuration.md](architecture/configuration.md) | Configuration, feature flags |

## Reliability

| File | Read when |
|---|---|
| [reliability/resilience.md](reliability/resilience.md) | Timeouts, retries, circuit breakers, rate limiting, graceful degradation |
| [reliability/consistency.md](reliability/consistency.md) | Idempotency, concurrency control, distributed locks, Outbox, Saga |
| [reliability/operations.md](reliability/operations.md) | Health checks, graceful startup and shutdown, SLOs, capacity |
| [reliability/performance.md](reliability/performance.md) | Performance profiling, connection pools, load testing |

## Observability

| File | Read when |
|---|---|
| [observability/logging.md](observability/logging.md) | Log fields, levels, redaction |
| [observability/metrics-and-alerts.md](observability/metrics-and-alerts.md) | Metrics and alerts |
| [observability/tracing.md](observability/tracing.md) | Distributed tracing |

## Data

| File | Read when |
|---|---|
| [data/schema-design.md](data/schema-design.md) | Designing table schemas |
| [data/indexes-and-queries.md](data/indexes-and-queries.md) | Indexes, SQL, slow queries |
| [data/transactions.md](data/transactions.md) | Transactions, locks, deadlocks |
| [data/migrations.md](data/migrations.md) | Database migrations |
| [data/data-access.md](data/data-access.md) | model/repo entry points, ORM boundaries |
| [data/cache-and-redis.md](data/cache-and-redis.md) | Caching and Redis |
| [data/document-and-search.md](data/document-and-search.md) | MongoDB and Elasticsearch |

## Async

| File | Read when |
|---|---|
| [async/messaging.md](async/messaging.md) | Message queues |
| [async/background-jobs.md](async/background-jobs.md) | Scheduled jobs and background jobs |

## Security

| File | Read when |
|---|---|
| [security/authentication.md](security/authentication.md) | Login, sessions, JWT, OAuth |
| [security/authorization.md](security/authorization.md) | Permissions, object-level checks, multi-tenancy |
| [security/input-and-injection.md](security/input-and-injection.md) | Input validation and injection protection |
| [security/crypto-and-secrets.md](security/crypto-and-secrets.md) | Encryption, keys, TLS |
| [security/web-risks.md](security/web-risks.md) | CORS, CSRF, SSRF, security response headers |
| [security/file-handling.md](security/file-handling.md) | File upload and download |
| [security/privacy.md](security/privacy.md) | Personal information and privacy compliance |
| [security/audit-and-abuse.md](security/audit-and-abuse.md) | Audit logs, rate limiting, abuse prevention |

## AI applications

| File | Read when |
|---|---|
| [ai/agents.md](ai/agents.md) | Agents, tool calls |
| [ai/rag.md](ai/rag.md) | Retrieval-augmented generation |

## Testing

| File | Read when |
|---|---|
| [testing.md](testing.md) | Backend integration tests, migration tests, performance tests |
