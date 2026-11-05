# Domain modeling

This document explains when to use domain-driven design (DDD) and the basic rules when you do. For table schema design, see [Schema design](../data/schema-design.md).

## Scope

### Decide first whether you need DDD [Recommended]

The cost of DDD is more types, more mapping, and more discussion. It is worth it only when business rules are truly complex. Ubiquitous language and bounded contexts are cheap, and most systems benefit from them. For tactical patterns such as aggregates and domain events, decide by the table below.

| Situation | Approach |
|---|---|
| Many rules, complex state transitions, domain experts can stay involved | Use entities, value objects, and aggregates |
| Mostly CRUD, few rules | Plain layering is enough. Do not introduce aggregates or domain events |
| Technical services, such as log collection or file transcoding | Do not use DDD. Organize by data flow |
| Uneven complexity within one system | Use it only in complex modules and keep other modules simple |

**Why:** Applying tactical patterns to simple modules only adds mapping code and does not reduce defects.

**Source:** Fowler DDD

## Domain language

### Use business words in code [Recommended]

Names of types, functions, tables, and APIs match what business people say. Use one word for one concept, and do not share a word between different concepts. When a word works poorly, change it together with business people, then sync the change to the code.

**Why:** Software cannot handle ambiguity. When the business and the code use different terms, every conversation needs translation, and translation errors become defects.

**Example:**

| Usage | Verdict |
|---|---|
| The business says "refund", the code says `reverseTransaction` | Avoid |
| The business says "refund", the code says `refund` | Prefer |

**Source:** Fowler: Ubiquitous Language

### The same word can have different models in different contexts [Recommended]

Large systems cannot maintain one unified model. When "customer" has different attributes and rules in sales and in customer support, split it into two models. Do not build one big customer object with every field. Record each context's responsibilities, main terms, and mappings in the [project docs](../../general/project-docs.md).

**Why:** A forcibly unified model is pulled by several sets of requirements at once, and nobody can change it.

**Source:** Fowler: Bounded Context

## Entities and value objects

Keep tables consistent with the concept hierarchy. See [Schema design](../data/schema-design.md).

### Distinguish by whether identity must be tracked [Recommended]

| Type | Basis | Equality | Examples |
|---|---|---|---|
| Entity | Still the same thing after attributes change | By identifier | Order, account |
| Value object | Interchangeable when all attributes are identical | By all attributes | Amount, date range |

The same concept can be a different type in different systems. For example, an address is usually a value object in an e-commerce system and may be an entity in a mapping system.

Model as a value object by default. Upgrade to an entity only when identity must be tracked.

**Source:** Fowler: Value Object, Azure: Tactical DDD

### Make value objects immutable [Recommended]

To change a value object, create a new object. Do not modify the original.

**Why:** Mutable shared value objects cause aliasing problems: change one place and another changes with it.

**Source:** Fowler: Value Object, Azure: Tactical DDD

### Use value objects to carry validation and units [Recommended]

Express values with a format or unit, such as amounts and email addresses, as value objects, and validate them at creation. Do not pass raw numbers and strings around.

### Put business rules on the object that owns the data [Recommended]

When using a domain model, put state transitions and validation in the entity. An entity with only fields and accessors, with all rules in services, is an anemic model: you pay the mapping cost and get none of the model's benefits.

Keep application services thin. They only coordinate and delegate.

Simple modules that do not use a domain model are not bound by this rule. Transaction-script services are a reasonable choice there.

**Source:** Fowler: Anemic Domain Model, Azure: Tactical DDD

## Aggregates

### An aggregate is a consistency boundary [Recommended]

An aggregate groups objects that must satisfy invariants together, with one entity as the root. Changes go through the root, and the root guarantees the invariants.

**Why:** If internal objects are changed around the root, nobody is responsible for the invariants.

**Example:** An order and its order lines form one aggregate. The order, as the root, guarantees "the order total equals the sum of its lines". Outside code cannot change order lines directly.

**Source:** Fowler: DDD Aggregate, Azure: Tactical DDD

### Reference only the aggregate root from outside [Recommended]

Outside code does not hold references to objects inside an aggregate. When one aggregate references another, it stores only the other's identifier.

**Example:**

Avoid:

```json
{ "orderId": "1001", "customer": { "id": "7", "name": "Jane Doe", "level": 3 } }
```

Prefer:

```json
{ "orderId": "1001", "customerId": "7" }
```

**Source:** Fowler: DDD Aggregate, Azure: Tactical DDD

### Change only one aggregate per transaction [Recommended]

The aggregate is the unit of loading, saving, and transactions. Complete changes across aggregates through domain events or later steps in the application layer, and accept eventual consistency.

**Exceptions:** In a monolith, when several aggregates must succeed together and the business does not accept an intermediate state, they can share a transaction. In that case, first check whether the aggregate boundaries are wrong.

**Source:** Fowler: DDD Aggregate

### Keep aggregates small [Recommended]

Include only data that must be consistent within one transaction. Split objects with different lifecycles into different aggregates. Otherwise unrelated updates contend for the same lock.

**Source:** Azure: Tactical DDD

## Domain events

### Events express business facts that have happened [Recommended]

Name events in the past tense. Describe what happened in the business, not what happened in the database.

**Example:**

| Name | Verdict |
|---|---|
| `OrderRowUpdated` | Avoid: this is a technical operation |
| `DeliveryCanceled` | Prefer: this is a business fact |

**Source:** Azure: Tactical DDD

### Do not change event content [Recommended]

Do not change an event after it is created. When an original event is wrong, emit a new correcting event. When the distinction matters, record both the time the event occurred and the time the system learned of it.

**Source:** Fowler: Domain Event

### Use commands when you expect the other side to act [Recommended]

When the sender needs the receiver to get something done, send a command. Do not disguise it as an event. The sender of an event does not care who responds.

**Source:** Fowler: Event-Driven

### Watch for processes hidden in events [Recommended]

When events chain multiple business steps, the full process cannot be seen in the code and can only be observed at runtime. When there are many steps and overall control is needed, switch to explicit orchestration. See the Saga part of [Consistency](../reliability/consistency.md).

**Source:** Fowler: Event-Driven

### Consumers see only committed events [Required]

Events across modules or services become visible to consumers only after the transaction commits. For how to guarantee this, see the Outbox part of [Consistency](../reliability/consistency.md). For delivery and consumption rules, see [Messaging](../async/messaging.md).

**Source:** Azure: Tactical DDD

### Do not use event sourcing and CQRS by default [Recommended]

Event sourcing and CQRS are two independent patterns, and both add significant complexity. Introduce them only when auditing, history reconstruction, or differences between read and write models truly require it.

**Source:** Fowler: Event-Driven
