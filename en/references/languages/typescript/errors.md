# TypeScript / JavaScript error handling

For language-independent principles, see [general/error-handling.md](../../general/error-handling.md). This document covers only language mechanisms.

## Error mechanisms

### Throw only `Error` and its subclasses [Recommended]

Only `Error` has a stack. This also applies to `Promise.reject` and `AbortController.abort(reason)`. Always use `new Error()`.

**Example:**

Avoid:

```ts
throw 'order not found';
await Promise.reject({ code: 404 });
```

Prefer:

```ts
throw new OrderNotFoundError(orderId);
```

**Exceptions:** When a framework uses throwing for control flow, follow the framework, such as `throw redirect()`. A `catch` that catches such a value must rethrow it unchanged.

**Tooling:** `@typescript-eslint/only-throw-error`, `prefer-promise-reject-errors`.

**Source:** Google TypeScript Style Guide, typescript-eslint Docs

### Treat caught values as `unknown` [Recommended]

`strict` includes `useUnknownInCatchVariables`, so `catch` variables are `unknown`. Also type `.catch()` callback parameters as `unknown`. Check with `instanceof` before reading properties.

**Tooling:** `@typescript-eslint/use-unknown-in-catch-callback-variable`.

**Source:** TSConfig Reference, typescript-eslint Docs

### Return expected failures, throw the rest [Recommended]

Business failures the caller must handle can be returned as a discriminated union, so the type system forces handling. Throw for program defects, dependency failures, and failures this layer cannot decide on. Use one approach within a module.

### No empty `catch` [Required]

Catching and doing nothing swallows the error. When an error truly can be ignored, catch only that error, rethrow the rest, and write the reason in a one-line comment next to it.

**Tooling:** ESLint `no-empty`.

**Source:** Google TypeScript Style Guide

### Wrap only code that can throw in `try` [Recommended]

With an oversized `try` block you cannot see which line may fail, and it may catch unrelated errors.

**Source:** Google TypeScript Style Guide

### Use `return await` when you need to catch [Required]

`return promise` in `try` does not wait, so the rejection does not reach the current `catch`, and `finally` also runs early.

**Example:**

```ts
try {
  return await loadOrder(id);
} catch (error) {
  throw new OrderLoadError(id, { cause: error });
}
```

**Tooling:** `@typescript-eslint/return-await`.

**Source:** typescript-eslint Docs

### Release resources with `finally` or `using` [Required]

Release resources on error paths too. `using` and `await using` need runtime support and `Symbol.dispose`. Use `finally` when they are not supported.

**Source:** MDN

## Custom errors and their ownership

### Extend `Error` and set `name` on custom errors [Recommended]

Define a subclass only when callers need to branch by type. The constructor accepts `ErrorOptions` to support `cause`, and business fields are read-only properties.

**Example:**

```ts
export class OrderLoadError extends Error {
  override readonly name = 'OrderLoadError';
  readonly orderId: string;

  constructor(orderId: string, options?: ErrorOptions) {
    super(`Failed to load order orderId=${orderId}`, options);
    this.orderId = orderId;
  }
}
```

**Source:** Google TypeScript Style Guide

### Define error classes in the module that uses them [Required]

Do not build a global error table. Use a `code` field for checks across packages, because `instanceof` fails when the same package is loaded twice.

**Source:** Team

## Wrapping and checking

### Preserve the root cause with `cause` when rethrowing [Required]

State in the message what was being done and which object was involved. `cause` goes in the `options` object, the second argument. `Error` is assignable to `ErrorOptions`, so passing an error object directly as the second argument raises no error but loses the root cause.

**Example:**

Avoid:

```ts
throw new Error('Charge failed', error);
```

Prefer:

```ts
throw new PaymentError(`Charge failed orderId=${order.id}`, { cause: error });
```

**Source:** Google TypeScript Style Guide, MDN

### Check with `instanceof` or `code`, do not parse messages [Required]

Error messages are not a stable API. For Node.js built-in errors use `error.code`, such as `ENOENT`.

**Source:** Node.js API Docs

### Use `AggregateError` or per-item results for batch failures [Recommended]

Check `Promise.allSettled` results item by item. Do not just `await` and assume everything succeeded. When failures need to surface, collect them with `AggregateError`.

**Source:** MDN

## Logging

### Pass the error object to the logger, not just `message` [Recommended]

Logging only `message` loses the stack and `cause`. Use structured logging, such as Pino, with the error in a dedicated field.

**Example:**

```ts
logger.error({ err: error, orderId }, 'Charge failed');
```

**Source:** Express Docs

### Middle layers wrap, not log [Recommended]

For the principle, see [general/error-handling.md](../../general/error-handling.md). Middle layers wrap with `cause` and throw, and logging happens in one place: the framework error handler or the job entry point.

## Framework boundaries

### Convert errors in the framework error handler [Recommended]

**Applies to:** production service

Route handlers only throw business errors, and the mapping to HTTP status codes and response bodies lives in one place. Return a generic 500 for unrecognized errors, and keep stacks out of responses.

| Framework | Location | Note |
|---|---|---|
| Express 5 | Four-argument error middleware, registered last | Rejections from `async` handlers go to `next(err)` automatically. When the response has already started, hand off to `next(err)` |
| Fastify | `setErrorHandler` | Applies per encapsulation context |
| NestJS | Exception filters | Non-`HttpException` errors return 500 by default |

**Source:** Express Docs, Fastify Docs, NestJS Docs

### Catch errors in callbacks and events [Required]

`try` cannot catch errors in callbacks that run later. An `'error'` on `EventEmitter` with no listener throws and ends the process. Do not use `async` functions directly as event listeners or `void` callbacks: nobody handles their rejections.

**Tooling:** `@typescript-eslint/no-misused-promises`.

**Source:** Node.js API Docs, typescript-eslint Docs
