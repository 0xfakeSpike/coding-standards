# Backend testing

This document covers testing practices specific to the backend. For test layers and types of test doubles, see [testing-strategy.md](../general/testing-strategy.md). For test frameworks, see the language layer. For contract tests, see [contracts.md](api/contracts.md).

## Integration tests

### Write integration tests for API and storage logic [Recommended]

Cover request parsing, authorization, transactions, and error responses. These problems occur at module boundaries, and unit tests do not catch them.

**Example:**

| Scenario | Assertion |
|---|---|
| No token | 401, no data created |
| Accessing another user's resource | 403 or 404 |
| Invalid parameters | 4xx, error structure matches the contract |
| Failure partway through | Transaction rolls back, no data left behind |
| Duplicate request | Only one record created |

### Drive tests through public entry points [Recommended]

Send requests through the HTTP or RPC entry point and assert on the response and the persisted result. Do not call internal methods directly.

### Isolate test data [Recommended]

Each test case prepares its own data and isolates it with transaction rollback, a separate database, or unique business keys. Do not depend on data left by other test cases.

**Why:** Shared data makes tests depend on execution order and prevents running them in parallel.

**Source:** Fowler: Eradicating Non-Determinism

### Assert async results by polling with a timeout [Recommended]

Wait for the results of message consumption and background jobs by polling. Do not use fixed-length waits.

**Source:** Fowler: Eradicating Non-Determinism

## Testcontainers

### Test database logic against a real database [Recommended]

Use Testcontainers to start databases, caches, and message queues of the same type and version as production.

**Why:** The SQL dialect and transaction behavior of an in-memory database can differ from production.

**Source:** Testcontainers Docs

### Pin image tags [Recommended]

Image tags match production versions. Do not use `latest`.

**Example:**

Avoid:

```text
postgres:latest
```

Prefer:

```text
postgres:17.2-alpine
```

**Why:** Floating tags make test results change over time.

### Do not hardcode addresses and ports [Recommended]

Container ports map to random host ports, and the address may not be `localhost`. Read connection details after the container starts, then inject them into the application configuration.

**Why:** Fixed ports conflict with local services or parallel tests, and some CI environments cannot connect through `localhost`.

**Source:** Testcontainers Docs

### Configure a readiness wait strategy [Recommended]

A connectable port does not mean the service is ready. Choose a log, HTTP, or health check wait based on the dependency, and adjust the startup timeout as needed.

**Source:** Testcontainers Docs

### Share containers within a test suite [Recommended]

Containers start slowly. Start them only once for the whole test run and let Testcontainers clean them up. Isolate test cases by clearing data or using separate databases.

**Source:** Testcontainers Docs

### Do not use reusable containers in CI [Recommended]

**Applies to:** team project

Reusable containers are an experimental feature, only for speeding up local development. They do not stop after tests finish, and state can carry over to the next run.

**Source:** Testcontainers Docs

## Third-party test doubles

### Replace third-party services at the client layer [Recommended]

Business code depends on a client interface you define, and tests replace it. When you need to verify serialization and error mapping, test the client itself against a local HTTP test double.

**Tooling:** WireMock

### Cover failure scenarios in test doubles [Recommended]

Besides successful responses, cover timeouts, dropped connections, rate limiting, 5xx, and malformed responses, and assert the retry, graceful degradation, and error mapping behavior.

**Example:**

| Test double behavior | Expected |
|---|---|
| Delay exceeds the timeout | Fails on time, does not block |
| Returns 503 | Retries per policy, with a capped number of attempts |
| Returns 400 | No retry |
| Response is missing fields | Returns a clear error, writes no bad data |

### Do not call real third parties in automated tests [Required]

Rate limits, data changes, and network problems in real services cause instability and can also incur charges. Verify real connectivity with a small number of separately run smoke tests.

**Source:** Fowler: Eradicating Non-Determinism

### Keep test doubles consistent with real behavior [Recommended]

Test double data comes from real response samples or the other party's contract, and is checked with contract tests. Otherwise the double can diverge from the real service.

**Source:** Fowler: Eradicating Non-Determinism, Pact Docs

## Migration tests

CI runs all migrations on an empty database of the real type. See [migrations.md](data/migrations.md).

### Run integration tests on the schema built by migrations [Recommended]

Do not replace migrations with ORM auto-created tables or separately maintained SQL files.

**Why:** Otherwise tests verify a schema that differs from production.

### Verify that migration results match the code [Recommended]

After migrating, compare the actual schema with the model definitions, and fail CI on any difference.

### Verify data migrations with representative data [Recommended]

When a migration changes existing data, prepare data that includes nulls and boundary values, and assert the result after migration. For large table changes, measure duration and lock impact at close to production data volume.

## Performance tests

For load test environments, data, and load generation, see [Performance](reliability/performance.md).

### Define targets before testing [Recommended]

State the scenario, concurrency or arrival rate, data volume, and pass criteria, such as P99 under 300 ms and error rate under 0.1%. Without targets, results cannot be judged. For measurement methods, see [performance.md](../general/performance.md). For load testing and capacity management, see [performance.md](reliability/performance.md).

**Example:**

| Item | Value |
|---|---|
| Scenario | Create order |
| Load | 200 requests per second for 15 minutes |
| Data | 10 million orders |
| Pass criteria | P99 under 300 ms, error rate under 0.1% |

### Look at percentiles and error rates [Recommended]

Report P50, P95, P99, throughput, and error rate, not just the average. Also record CPU, memory, connection pools, and queue backlog to find the bottleneck.

### Exclude warm-up effects [Recommended]

Do not count the warm-up phase in results. Run long enough for caches and connection pools to reach a steady state.

### Compare against a baseline [Recommended]

Save every result and compare in the same environment. Run key endpoints regularly to catch degradation early.

### Do not load test production or third parties directly [Required]

Run load tests in an isolated environment and use test doubles for third-party dependencies. Test doubles must simulate real latency.

**Exceptions:** When load testing production is truly necessary, review it in advance, use isolated test data, and have a way to stop ready.
