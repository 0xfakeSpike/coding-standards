# Error handling principles

This document covers only language-independent principles. For specific mechanisms, see the error handling sections of the language layer.

## Basic requirements

### Surface errors immediately [Required]

Make an error visible as soon as you find it. Do not mask it with a default value or keep running in an invalid state.

**Why:** The closer an error is to its root cause, the easier it is to locate.

**Source:** Team

### Do not ignore errors [Required]

Silently dropped errors push the problem somewhere harder to debug. When an error truly can be ignored, write a one-line reason next to it.

### Do not let a local failure bring down the whole [Required]

In long-running services and batch jobs, when a single operation fails, isolate the impact and return useful information: where it failed, why, and what can be done next.

**Example:** When a tool call fails, return the failure reason to the caller as the result, and keep the process running.

**Boundary:** This is not swallowing errors. Errors must still be exposed to the caller and recorded. One-off scripts and operations that require atomicity terminate entirely on failure.

**Source:** Team

## Classification

### Distinguish error types [Recommended]

| Type | Example | Handling |
|---|---|---|
| Expected business failure | Insufficient balance | Return a clear result |
| Caller error | Invalid parameter | Reject and explain why |
| Environment or dependency failure | Network outage | Handle per policy and record it |
| Program defect | Invariant violated | Surface immediately and fix the code |

### Define errors in the module that uses them [Required]

Each module defines the errors it needs. Do not keep one large error table shared by all modules.

**Source:** Team

## Propagation

### Keep the root cause and add context [Required]

When passing an error up, state what was being done and which object was being handled, and keep the original error.

### Handle errors at the layer that can decide [Recommended]

If you cannot handle it, pass it up.

### Record each error only once [Recommended]

Recording at every layer produces duplicate logs that get in the way of judgment.

### Convert errors at the boundary [Recommended]

Convert low-level details into this module's errors at the module boundary. Do not expose third-party error types to the business layer.

## Retries

### Do not retry by default [Recommended]

Consider a retry only when the failure is truly temporary and the operation can be safely repeated. Retries add complexity, so first decide whether they are worthwhile.

## User-visible messages

### Do not expose internal details in external messages [Required]

Users see an understandable reason and next step, not stack traces or internal identifiers.

## Resources

### Release resources on error paths too [Required]

Use the mechanism the language provides. Do not rely on releasing manually in each branch.
