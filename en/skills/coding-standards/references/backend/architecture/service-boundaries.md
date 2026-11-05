# Service boundaries

This document sets rules for how modules and services are divided, how they communicate, and who owns data. For layering inside a module, see [layering.md](layering.md). For general principles of dividing responsibilities, see [Responsibilities](../../general/responsibilities.md).

## Component responsibilities

### Keep module boundaries clear and responsibilities single, without overreach [Required]

Every module can state what it is responsible for in one sentence. A module does only what that sentence covers. It does not do work for other modules or reach into their internals.

**Why:** Once boundaries blur, the impact of a change cannot be judged, and modules cannot be understood or tested on their own.

**Example:** A notification module is responsible only for sending messages. "When to notify" is decided by the triggering business module and is not written into the notification module.

**Source:** Team

### Each module is responsible for its own state [Recommended]

A module does not rely on other modules to finish its work along the way. It checks and triggers what it needs itself. See [Responsibilities](../../general/responsibilities.md).

### Divide by business capability, not technical layer [Recommended]

Organize modules and services around business capabilities, such as orders and payments. Do not split them into technical slices such as a "data service" or "validation service". A module does not span bounded contexts. When a module mixes two domain models, adjust the models first.

**Why:** With technical slicing, one requirement changes several modules at once and needs several teams to coordinate.

**Source:** Fowler: Microservices, Azure: Microservice Boundaries

## Modular monolith

### Prefer a modular monolith [Recommended]

Build new systems as one deployment unit by default, with strict internal modules. Split only after boundaries are stable and there is a clear need.

**Why:** Boundaries are hard to draw correctly early on, and moving functionality between services costs far more than moving it between modules. Microservices also carry the extra cost of distributed calls, eventual consistency, and operations.

**Exceptions:** The team already has mature microservice experience, and the boundaries have been proven by an existing system.

**Source:** Fowler MonolithFirst, Fowler: Microservice Premium

### Communicate between modules only through public interfaces [Recommended]

A module exposes a clear interface, and everything else is internal implementation. Other modules do not reference internal types or access internal tables.

**Why:** A monolith has no process isolation, and boundaries that rest only on convention gradually break down.

**Tooling:** Use a dependency checking tool in CI to forbid cross-module references to internal directories.

**Source:** Fowler: Microservice Trade-Offs

### Isolate module data too [Recommended]

In a monolith, each table belongs to exactly one module. Other modules that need the data call that module's interface. Do not JOIN across modules.

**Exceptions:** Put cross-business list and report queries in a dedicated read-only module that only queries and never writes. With large data volumes, consider read replicas or summary tables.

**Source:** microservices.io: Database per Service

## Splitting services

### Have a specific reason to split [Recommended]

Split only when at least one of the following holds and a modular monolith can no longer solve it:

| Reason | How to judge |
|---|---|
| Independent scaling | Resource needs differ greatly, and scaling everything is clearly wasteful |
| Team boundaries | Several teams block each other in one deployment unit |
| Fault isolation | Failures in one part must not affect the core flow |
| Independent release cadence | Release frequencies differ greatly, and combined releases are costly |
| Security or compliance isolation | Sensitive data needs its own access boundary |

"Microservices are more advanced" and "the codebase got big" are not reasons.

**Why:** Remote call failures, eventual consistency, and operational burden from distribution are certain costs, and the benefit must exceed them.

**Source:** Fowler: Microservice Premium, Fowler: Microservice Trade-Offs

### Check the split result [Recommended]

A split plan must pass these checks:

- Each service has a single responsibility, and one small team can maintain it independently.
- Services do not make frequent fine-grained calls to each other.
- A service can be deployed alone, without releasing other services at the same time.
- No strong consistency requirement has been split across two services.

When two services are always changed together and call each other often, they should be merged.

**Source:** Azure: Microservice Boundaries

## Data ownership

### Give each piece of data one owner [Recommended]

Only the owning service can write its data. Other services get it through APIs or events.

**Why:** When several services write the same table, any schema change needs cross-team coordination, and they also interfere with each other at runtime.

**Tooling:** Enforce boundaries with separate database accounts and grants, not by convention.

**Source:** microservices.io: Database per Service, microservices.io: Shared Database

### Choose the level of isolation as needed [Recommended]

| Approach | Cost | Use when |
|---|---|---|
| Private tables | Lowest | Few services sharing one instance |
| Separate schema | Low | Ownership needs to be clearer |
| Separate database instance | High | High throughput or resource isolation needed |

**Source:** microservices.io: Database per Service

### Have an explicit approach for cross-service queries [Recommended]

| Approach | Use when | Cost |
|---|---|---|
| Caller combines API results | Small data volume, high freshness requirement | Multiple calls, depends on the availability of several services |
| Subscribe to events and maintain a local read-only copy | Frequent queries, delay is acceptable | Redundant data, delay must be handled |

For cross-service writes, see [Consistency](../reliability/consistency.md).

**Source:** microservices.io: Database per Service, Fowler: Event-Driven

## Service communication

### Choose by whether the caller needs an immediate result [Recommended]

| Approach | Use when | Risk |
|---|---|---|
| Synchronous calls (HTTP, gRPC) | The caller must get the result to continue | Availability and latency compound at each level |
| Asynchronous messages | The caller does not need to wait for the result | Duplicates, out-of-order messages, and backlog must be handled |

**Source:** Azure: Interservice Communication

### Avoid long synchronous call chains [Recommended]

Keep synchronous call chains on the user request path as short as possible. Make parts that need not finish immediately asynchronous, or redraw the boundaries.

**Why:** Each extra hop adds latency, and if any link in the chain is unavailable, the whole request fails.

**Source:** Fowler: Microservices, Azure: Interservice Communication

### Use coarse-grained interfaces for remote calls [Recommended]

Remote calls are far more expensive than in-process calls. Complete one whole piece of work per call, and do not expose local object accessors directly as remote interfaces. Keep business logic inside services. Message middleware only delivers.

**Source:** Fowler: Microservices, Azure: Chatty I/O

### Treat every remote call as able to fail [Required]

For timeout, retry, and circuit breaker rules, see [Resilience](../reliability/resilience.md). For API design, see [REST API](../api/rest.md) and [gRPC](../api/grpc.md). For messaging, see [Messaging](../async/messaging.md).

## API gateway

### Gateways do only cross-cutting work [Recommended]

| Fits in the gateway | Keep out of the gateway |
|---|---|
| Routing, TLS termination | Business rules |
| Identity validation, client rate limiting | Business state decisions |
| Request logging, simple response aggregation | Cross-service transaction orchestration |

**Why:** The gateway is the shared path of all requests. With business logic in it, it becomes a new monolith that every team has to change.

**Source:** Azure: API Gateway, Fowler: Microservices

### The gateway is neither the only line of defense nor a single point of failure [Required]

**Applies to:** production service

After the gateway validates identity, services must still check permissions internally. See [Authorization](../security/authorization.md). Do not expose backend services directly to the public internet. Deploy the gateway with multiple instances.

**Source:** Azure: Gateway Routing

### Do not introduce a gateway when there are few services [Recommended]

With only one or two services, a gateway only adds a hop and something else to operate. Consider a separate gateway for each client type (BFF) only when needs differ greatly between client types.

**Source:** Azure: Gateway Routing, microservices.io: API Gateway

## Anti-patterns

### Recognize common anti-patterns [Recommended]

| Anti-pattern | Signal | Fix |
|---|---|---|
| Distributed monolith | One requirement needs several services released together | Redraw boundaries or merge |
| Shared database | Several services read and write the same table | Name an owner, others go through its API |
| Chatty calls | Completing one operation takes many back-and-forth calls | Merge services or switch to coarse-grained interfaces |
| Splitting services by technical layer | Data services exist that only do CRUD | Reorganize by business capability |
| Shared business library | Several services depend on a shared library containing business rules | Move the rules back into the owning service |
| Do-everything gateway | Business decisions are written in gateway configuration or scripts | Move them back into the owning service |

**Source:** Azure: Microservice Boundaries, microservices.io: Shared Database
