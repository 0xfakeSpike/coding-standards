# GraphQL standards

This document covers server-side GraphQL design. For the authorization model, see [authorization.md](../security/authorization.md). For N+1 on the database side, see [indexes-and-queries.md](../data/indexes-and-queries.md).

## Schema design

### Design the schema for client use [Recommended]

The schema reflects business concepts and client scenarios. Do not copy table structures. Name Mutations after specific actions, such as `updateHumanName`, so parameters can be declared non-null and constrained directly by the schema.

**Source:** GraphQL Docs

### Decide nullability per field [Recommended]

Fields are nullable by default. Declare `!` only when you are sure the field cannot fail. When a non-null field errors, `null` propagates up to the nearest nullable parent field, and one small error can wipe out a whole block of data.

**Example:**

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  shippingEstimate: ShippingEstimate
}
```

**Why:** `shippingEstimate` depends on an external service. When it fails, the whole order should not become `null`.

**Source:** GraphQL Docs

### Put writes only in top-level Mutation fields [Required]

Query fields and nested fields have no side effects. Use an `input` type for parameters and return the modified object so the client gets the latest state.

**Example:**

```graphql
input CreateReviewInput {
  stars: Int!
  commentary: String
}

type Mutation {
  createReview(orderId: ID!, input: CreateReviewInput!): Review
}
```

**Source:** GraphQL Docs

### Do not treat multiple Mutations as a transaction [Required]

Top-level Mutations in one request run serially, but a failure partway through does not roll back fields already completed. When you need atomicity, design one Mutation and open a transaction in the business layer.

**Source:** GraphQL Docs

### Put authorization in the business logic layer [Required]

Complete authentication before GraphQL execution and put the identity in the request context. Put permission checks in the business layer, not in individual resolvers. Otherwise, some access paths to the same data miss the check. Do object-level checks on every node, not only on entry fields.

**Source:** GraphQL Docs, OWASP GraphQL Cheat Sheet

## N+1 and DataLoader

### Batch-load related fields [Recommended]

**Applies to:** production service

When resolvers query related data one record at a time, one request produces many downstream calls. Use DataLoader to merge keys from the same execution cycle into one batch query.

**Example:**

Avoid:

```sql
SELECT id, name FROM users WHERE id = 1;
SELECT id, name FROM users WHERE id = 2;
SELECT id, name FROM users WHERE id = 3;
```

Prefer:

```sql
SELECT id, name FROM users WHERE id IN (1, 2, 3);
```

**Source:** GraphQL Docs

### Create DataLoader instances per request [Required]

Results returned by the batch function match the input keys one-to-one in order and count. Missing keys return empty values.

**Why:** When shared across requests, the cache can return one user's data to another user and can also serve stale data.

## Pagination

### Use Connections for lists that can get large [Recommended]

Follow the Cursor Connections specification. A connection type contains `edges` and a non-null `pageInfo`. An edge type contains `node` and `cursor`. Page forward with `first` and `after`, and backward with `last` and `before`. Set upper limits on `first` and `last`.

**Example:**

```graphql
type OrderConnection {
  edges: [OrderEdge]
  pageInfo: PageInfo!
}

type OrderEdge {
  cursor: String!
  node: Order
}

type PageInfo {
  hasPreviousPage: Boolean!
  hasNextPage: Boolean!
  startCursor: String
  endCursor: String
}

type User {
  orders(first: Int, after: String): OrderConnection
}
```

**Source:** GraphQL Cursor Connections, GraphQL Docs

### Keep cursors opaque and ordering stable [Recommended]

Clients do not parse cursors. Keep ordering consistent across pages. `last` and `before` do not reverse the order of edges.

**Source:** GraphQL Cursor Connections

## Query complexity

### Limit query size before execution [Required]

**Applies to:** high risk

One query can trigger any number of downstream calls. Reject oversized requests before execution.

| Limit | Description |
|---|---|
| Depth | Limit nesting levels, with stricter limits for nested lists |
| Breadth | Limit the number of top-level fields and aliases |
| Batching | Limit the number of operations in a single HTTP request |
| Cost | Assign field weights, estimate total cost, and reject requests over the limit |
| Timeout | Set timeouts at the application or infrastructure layer |

**Source:** GraphQL Docs, OWASP GraphQL Cheat Sheet

### Rate limit by query cost [Recommended]

Counting HTTP requests alone does not reflect the real cost. Deduct quota by cost in the business layer. Aliases and batching can pack many login attempts into one request, so rate limit sensitive operations by operation count.

**Source:** GraphQL Docs, OWASP GraphQL Cheat Sheet

### Use trusted documents for first-party clients [Recommended]

Register query documents at build time, and at runtime accept only hashes of registered documents. This does not work for public APIs.

**Source:** GraphQL Docs

### Disable introspection and debug UIs outside development [Recommended]

Disable introspection for services that serve first-party clients. This only makes probing harder. It does not replace authorization or query limits.

**Exceptions:** Public APIs can leave introspection open.

**Source:** GraphQL Docs, OWASP GraphQL Cheat Sheet

## Errors and transport

### Return errors through errors with a stable identifier [Recommended]

Field errors carry `message` and `path`. Put the machine-readable identifier in `extensions.code`. Clients do not parse `message` and must handle partial responses where `data` and `errors` are both present.

**Example:**

```json
{
  "data": { "order": null },
  "errors": [
    {
      "message": "Order not found",
      "path": ["order"],
      "extensions": { "code": "ORDER_NOT_FOUND" }
    }
  ]
}
```

**Source:** GraphQL Docs

### Hide error details in production [Required]

Convert all unknown resolver exceptions into a generic message. Also disable field suggestions in validation errors. Otherwise the schema can be inferred even with introspection off.

**Source:** GraphQL Docs, OWASP GraphQL Cheat Sheet

### Use correct HTTP semantics [Required]

The server supports POST. GET can be used only for queries, and Mutations must use POST. Return 2xx when `data` is non-null. When a request fails validation, usually return 400.

**Source:** GraphQL Docs
