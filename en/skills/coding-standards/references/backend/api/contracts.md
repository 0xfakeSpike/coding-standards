# API contracts

This document covers writing, managing, and verifying API contracts. For REST design, see [rest.md](rest.md). For test layers, see [testing-strategy.md](../../general/testing-strategy.md).

**Applies to:** team project (all rules in this document)

## OpenAPI first

### Give APIs a machine-readable contract [Recommended]

Describe HTTP APIs with OpenAPI and store the description in the code repository. When the contract is generated from code, validate it in CI so the docs do not go stale.

**Source:** Zalando RESTful API Guidelines

### Write the contract before the implementation [Recommended]

Define the contract and have callers review it before writing the implementation, so frontend, backend, and services can develop in parallel.

**Exceptions:** Internal APIs with a single caller can generate the contract from code.

**Source:** Zalando RESTful API Guidelines

### Keep one source of truth for the contract [Recommended]

Choose either a handwritten contract or code generation. Do not maintain two descriptions by hand at the same time.

### Give each operation a unique operationId [Recommended]

`operationId` is unique across the whole contract, case-sensitive, and follows common programming naming conventions. Code generators usually name methods after it.

**Source:** OpenAPI Specification

### Declare success and error responses [Recommended]

Each operation lists all success responses and the error responses callers need to handle differently. Error responses reference a shared error schema.

**Example:**

```yaml
paths:
  /orders/{id}:
    get:
      operationId: getOrder
      parameters:
        - $ref: "#/components/parameters/OrderId"
      responses:
        "200":
          description: Order details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"
        "404":
          $ref: "#/components/responses/Problem"
```

**Source:** OpenAPI Specification, Zalando RESTful API Guidelines

## Schema management

### Put shared structures in components [Recommended]

Define schemas, parameters, and responses used in multiple places in `components` and reference them with `$ref`. Structures used only once do not need extracting.

**Source:** OpenAPI Specification

### Write complete constraints in schemas [Recommended]

State `required`, `format`, `enum`, value ranges, and length limits. Do not use an unconstrained `object` for a known structure.

**Example:**

Avoid:

```yaml
Order:
  type: object
```

Prefer:

```yaml
Order:
  type: object
  required: [id, status, createdAt]
  properties:
    id:
      type: string
    status:
      type: string
      enum: [pending, paid, cancelled]
    createdAt:
      type: string
      format: date-time
```

**Why:** With incomplete constraints, generated code and validation provide no protection.

### Use JSON Schema syntax for nullable fields [Recommended]

Write nullable as `type: [string, "null"]`, not `nullable`. This requires OpenAPI 3.1 or later.

**Source:** OpenAPI Specification, Zalando RESTful API Guidelines

### Run static checks on contracts in CI [Recommended]

Validate the syntax, and use a ruleset to check naming, error responses, and pagination conventions.

**Tooling:** Spectral, Redocly CLI

## Generated code

### Do not hand-edit generated code [Required]

When adjustments are needed, change the contract or generator configuration, then regenerate.

**Why:** Manual edits are lost on the next generation.

### Make generation repeatable [Recommended]

Pin the generator version and generate with one command. When generated output is committed, CI regenerates it and checks that there is no diff.

**Tooling:** `git diff --exit-code`

### Keep generated types at the API boundary [Recommended]

Use generated types only in the API layer. Convert them to domain types before they enter the business layer.

## Contract tests

### Choose the verification method by scenario [Recommended]

| Scenario | Method |
|---|---|
| Consumers and providers are both built by teams in the organization, and the number of consumers is manageable | Consumer-driven contract tests, such as Pact |
| Public API, consumers cannot be identified one by one | Validate real responses with OpenAPI |
| The other team does not participate, or you cannot control provider data | Validate with OpenAPI, or maintain a third-party test double |
| API that only passes requests through without validating content | Establish the contract between the consumer and the downstream service |

**Source:** Pact Docs

### Contract tests do not replace functional tests [Recommended]

Contract tests only verify that both sides understand requests and responses the same way. Validation rules, side effects, and business logic are covered by the provider's own tests. Keeping one representative scenario for each class of error response is enough.

**Why:** If business rules are written into the contract, changing a rule on the provider side breaks consumer tests that were otherwise unaffected.

**Source:** Pact Docs

### Consumers assert only what they use [Recommended]

Match response fields by type by default. Use exact matching only when the content truly affects the consumer. Do not assert on unused fields or the provider's format rules.

**Source:** Pact Docs

### Consumer tests cover the real client code [Recommended]

The test scope is the client module that makes the call. Do not go through the UI, and do not test only raw HTTP calls. All calls in business code go through this module.

**Source:** Pact Docs

### Echo updated content in write responses [Recommended]

Each interaction is verified independently, so the test still passes when the provider ignores a misspelled field. Include the updated values in the response, or share test data between the read and write sides. The same applies to query parameters: the response can echo the parameters actually applied.

**Source:** Pact Docs

### Run provider verification on a local instance [Recommended]

Verify against a provider started locally or in CI, running alongside the other tests, not against a deployed environment. Downstream dependencies can be replaced, but replace only the layers after request parsing and validation.

**Why:** Running against a deployed environment gives slow feedback, is hard to debug, and cannot run in parallel.

**Source:** Pact Docs
