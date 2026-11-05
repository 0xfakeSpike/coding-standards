# TypeScript / JavaScript testing

For test strategy, see [general/testing-strategy.md](../../general/testing-strategy.md). This document covers only tool usage. Vitest does not check types at runtime. For type-checking requirements, see [project.md](project.md).

## Framework

### Default to Vitest [Recommended]

Use Vitest in new projects. It supports TypeScript and ESM directly. Projects already on Jest keep it, and one repository does not mix two frameworks. Small tools with no dependencies can use the built-in `node:test`.

**Source:** Vitest Docs, NestJS Docs

### Use type tests for type constraints [Recommended]

Write tests for public generics and utility types in `*.test-d.ts`, assert with `expectTypeOf`, and run them with `vitest --typecheck`. Use `@ts-expect-error` where an error is expected.

**Source:** Vitest Docs

## Organization

### Keep test files next to source [Recommended]

Tests for `order-service.ts` go in `order-service.test.ts` in the same directory. Put integration tests in `test/` or a separate Vitest project, so unit tests can run on their own.

**Source:** Vitest Docs

### Reset module-level state in beforeEach [Recommended]

For test independence requirements, see [testing-strategy.md](../../general/testing-strategy.md). Each test prepares its own data, and module-level mutable state is reset in `beforeEach`. Turning on `--no-isolate` for speed requires that tests leave no side effects behind.

**Source:** Vitest Docs

### Await async assertions [Required]

`expect(promise).rejects` must be awaited, or a failed assertion does not fail the test. Do not assert exceptions with `try/catch` plus `fail()`.

**Example:**

Avoid:

```ts
test('throws when the order does not exist', () => {
  expect(loadOrder('missing')).rejects.toThrow(OrderNotFoundError);
});
```

Prefer:

```ts
test('throws when the order does not exist', async () => {
  await expect(loadOrder('missing')).rejects.toThrow(OrderNotFoundError);
});
```

**Tooling:** `@typescript-eslint/no-floating-promises` covers test files.

**Source:** Google TypeScript Style Guide

### Do not commit `.only` [Required]

**Applies to:** team project

`.only` causes the other tests to be skipped. Vitest forbids `.only` in CI by default. Do not turn on `allowOnly`.

**Source:** Vitest Docs

### Use the context `expect` in concurrent tests [Recommended]

In `test.concurrent`, use the `expect` from the test context for snapshots and assertions, or results may be attributed to the wrong test.

**Source:** Vitest Docs

## Parameterization

### Use `test.each` or `test.for` for the same logic with multiple inputs [Recommended]

Put the input and expected result in the test case name. `test.each` spreads array arguments. `test.for` does not, and it can access the test context.

**Example:**

```ts
test.each([
  { totalCents: 0, expected: 'free' },
  { totalCents: 4999, expected: 'standard' },
  { totalCents: 5000, expected: 'free' },
])('$totalCents cents maps to $expected shipping', ({ totalCents, expected }) => {
  expect(selectShipping(totalCents)).toBe(expected);
});
```

**Source:** Vitest Docs

## Test doubles

### Prefer injected dependencies over module doubles [Recommended]

When a double can be passed as a parameter, do not use `vi.mock`. `vi.mock` is hoisted to the top of the file, runs before all imports, depends on module paths, and breaks easily during refactoring.

**Source:** Vitest Docs

### Restore double state for each test [Recommended]

Configure `restoreMocks: true`, or call `vi.restoreAllMocks()` in `afterEach`. An unrestored `vi.spyOn` pollutes later tests.

**Exceptions:** `restoreMocks` may affect concurrent tests in progress. Restore manually in those files.

**Source:** Vitest Docs

### Control time with fake timers [Recommended]

Do not write real `sleep`s. Use `vi.useFakeTimers()` and `vi.setSystemTime()`, which also replaces `Date`. Call `vi.useRealTimers()` in `afterEach`.

**Example:**

```ts
beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date('2026-01-01T00:00:00Z'));
});

afterEach(() => {
  vi.useRealTimers();
});
```

**Source:** Vitest Docs

### Intercept HTTP dependencies with MSW [Recommended]

Do not replace `fetch` itself. MSW intercepts at the network layer, so application code needs no changes. Configure undeclared requests to error, so tests do not reach the real network.

**Source:** Vitest Docs

### Keep double types complete [Recommended]

Preserve types with `vi.fn<typeof sendMail>()` and `vi.mocked()`. When a signature changes, the compiler catches stale doubles. When `any` is unavoidable, keep it local within the test file.

**Source:** Google TypeScript Style Guide

## Integration tests

### Test HTTP endpoints without a real port [Recommended]

Use `app.inject()` for Fastify and Supertest for Express and NestJS. This requires the app instance to be separate from `listen`.

**Example:**

```ts
const response = await app.inject({ method: 'GET', url: '/orders/1' });

expect(response.statusCode).toBe(200);
```

**Source:** Fastify Docs, NestJS Docs

### Use a real database instance [Recommended]

Verify SQL and transaction behavior with a real database in a container, not an in-memory substitute. Isolate each test with a transaction rollback or a separate schema.

### Release resources when tests end [Recommended]

Close the app, connection pools, and timers in `afterAll`. When the process will not exit, investigate with `process.getActiveResourcesInfo()`. Do not hide the problem with a forced exit.

**Source:** Node.js API Docs

## Benchmarks

### Use Vitest `bench` for benchmarks [Recommended]

Name files `*.bench.ts`. Compare results only within the same environment. Do not compare a development machine with CI.

**Source:** Vitest Docs

### Use the result in benchmark functions [Recommended]

Code with no observable side effects may be optimized away by the engine, leading to wrong conclusions. Return or consume the computed result.

**Source:** Vitest Docs

### Do not make benchmark results a hard gate [Recommended]

CPU load and GC both affect results. When assertions are needed, set a relative difference tolerance.

**Source:** Vitest Docs
