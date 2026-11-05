# Layered architecture

This document sets rules for backend code layering, dependency direction, and data conversion between layers. Layer names vary by language and framework. What matters is the division of responsibilities. For boundaries between modules, see [service-boundaries.md](service-boundaries.md). For domain models, see [domain-modeling.md](domain-modeling.md).

## Choosing placement

### Put each change where the responsibility best fits [Required]

Before you start, assess which layer and which module this logic best belongs to. Do not put it here just because "it can be done here right now".

**Why:** Logic stuffed into the nearest spot gradually mixes responsibilities, and every later change must first work out why it is there.

**Example:** When an API response is missing a field, do not add a quick query in the controller to fill it in. First decide which business module the data belongs to, then have that module provide it.

**Source:** Team

### Layer by responsibility, with one-way dependencies [Recommended]

| Layer | Responsibility | Does not |
|---|---|---|
| API layer (Controller, Handler) | Parses requests, validates parameter format, calls the application layer, converts responses | Write business rules or access the database directly |
| Application layer (Service, Use Case) | Orchestrates use cases, sets transaction boundaries | Depend on transport-layer objects such as HTTP |
| Domain layer (entities, domain services) | Business rules and invariants | Depend on frameworks, databases, or external systems |
| Data access layer (Repository, DAO) | Reads and writes data | Make business decisions |
| External clients (Client, Gateway) | Wraps third-party calls | Expose third-party types to business code |

When the business is simple, the application and domain layers can merge, and a layer that only forwards calls should be removed. But business rules must not sink into the data access layer or rise into the API layer.

**Why:** Once layers are mixed, business rules cannot be tested apart from the framework and database, and the same rule tends to be written separately at multiple entry points.

**Tooling:** Use a dependency checking tool in CI to validate references between layers.

**Source:** Martin: Clean Architecture

## Business logic and the database

Keep business logic out of the database. See [Schema design](../data/schema-design.md).

### Do not do presentation processing in the database [Recommended]

Move string concatenation, format conversion, and complex calculations to the application layer. Databases are hard to scale horizontally, and spending compute on this work squeezes read and write capacity. Keep aggregation, filtering, and sorting, which databases do well, in the database.

**Source:** Azure: Busy Database

## Dependency direction

### Point dependencies at the business core [Recommended]

Source dependencies point only from outer layers to inner layers. Business code does not reference framework types, database drivers, or third-party SDKs. When business code needs an external capability, it defines the interface it needs, and an outer layer implements it.

**Why:** When dependencies run the other way, replacing a database or third-party service affects business rules.

**Example:**

| Usage | Verdict |
|---|---|
| Domain layer references ORM row objects | Avoid |
| Application layer receives the framework's request object | Avoid |
| Application layer declares a "payment gateway" interface, and an external client implements it | Prefer |

**Source:** Martin: Clean Architecture, Cockburn: Hexagonal Architecture

### Pass dependencies in from outside [Recommended]

Business code does not create database connections or external clients directly. Dependencies come in through the framework's dependency injection or constructor parameters, and assembly happens in one place at the program entry point.

**Why:** Dependencies created internally cannot be replaced in tests, and timeouts and connection pools cannot be configured uniformly.

### Isolate the outside world with ports [Recommended]

Hexagonal architecture splits a system into inside and outside. The inside is the application. The outside is users, tests, databases, and third-party systems.

| Concept | Meaning | Examples |
|---|---|---|
| Port | A purposeful conversation defined by the application | "Save order", "Send notification" |
| Driving adapter | Converts external input into calls to the application | HTTP handlers, message consumers, test cases |
| Driven adapter | Converts application calls into specific technology | SQL implementation, email service implementation |

The criterion is whether a business use case can run without starting an HTTP server or database. Name ports by business purpose, not by technology.

What clean architecture and hexagonal architecture share is the requirement that the business core knows nothing of external technology. The number and names of rings are not hard requirements. When an external dependency has only one implementation and will not change soon, you can use the concrete implementation directly. See [General coding principles](../../general/principles.md).

**Source:** Cockburn: Hexagonal Architecture, Martin: Clean Architecture

## Data mapping

### Do not return database entities directly as API responses [Required]

API requests and responses use separately defined DTOs or schemas, converted in the API layer.

**Why:** Internal fields can leak, and the API shape gets locked to the table structure.

**Example:**

Avoid:

```json
{
  "id": 1001,
  "password_hash": "$2b$12$abc",
  "deleted_flag": 0,
  "internal_note": "High-risk user"
}
```

Prefer:

```json
{
  "id": "1001",
  "displayName": "Jane Doe"
}
```

### Pass simple data structures across layers [Recommended]

The inner layer decides the form of data crossing a layer boundary. Do not pass row objects from database frameworks or response objects from third-party SDKs inward. Convert them to inner-layer types in adapter code.

**Source:** Martin: Clean Architecture

### Mapping only converts [Recommended]

DTO mapping handles only field correspondence and format conversion. It makes no business decisions and issues no queries. Keep conversions for the same object in one place.

## Error boundaries

### Distinguish business errors from system errors [Recommended]

| Type | Examples | HTTP status | Log level |
|---|---|---|---|
| Business error | Insufficient balance, invalid parameter | 4xx | INFO or WARN |
| System error | Database unavailable, code defect | 5xx | ERROR |

For general principles, see [Error handling principles](../../general/error-handling.md). For response structure, see [REST API](../api/rest.md).

### Convert errors at a single entry point [Recommended]

Business code only throws or returns its own module's errors. A global error handler or middleware converts them to responses. Do not repeat this in each handler. For unknown errors, return a generic message externally and record the full context internally.

## Directory template

### Organize directories by business module first [Recommended]

Divide the top level by business module, then by layer inside each module. Do not put all controllers or all services in one top-level directory each.

**Why:** Requirements usually change along business modules. Organizing by module keeps a change in one directory and keeps module boundaries visible.

**Example:**

```text
src/
├── order/
│   ├── api/
│   ├── application/
│   ├── domain/
│   └── infrastructure/
├── payment/
│   ├── api/
│   ├── application/
│   ├── domain/
│   └── infrastructure/
└── main/
```

| Directory | Content |
|---|---|
| `api` | Handlers, DTOs, mapping |
| `application` | Use cases and transaction boundaries |
| `domain` | Entities, value objects, domain services, ports |
| `infrastructure` | Data access implementations, external clients |
| `main` | Program entry point and dependency assembly |

Small services can omit the subdirectories inside modules. For each language's directory conventions, see the project structure file in the language layer.
