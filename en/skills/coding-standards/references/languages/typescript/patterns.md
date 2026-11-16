# TypeScript design patterns

This file covers the idiomatic forms of design patterns in TypeScript. For when to use which pattern, see [design-patterns.md](../../general/design-patterns.md). The overall principle: do not write a class when functions and plain objects can express it. See "Use interfaces, not classes, for plain data" in [types.md](types.md) and "No pass-through functions or pointless wrappers" in [idioms.md](idioms.md).

## Structure

### Do not define an interface for a single implementation [Recommended]

Do not write `IUserService` plus `UserServiceImpl`. Types are structural, so a test double only needs the same shape to be passed in, and the implementation does not need to declare an interface up front. When there are several implementations, or a consumer uses only a few of the methods, the consumer declares the shape it needs.

### Expose fields directly instead of writing getters and setters [Recommended]

Use `readonly` for fields that do not change. Use `get` and `set` accessors or methods only when you need to compute, validate or keep an invariant.

### Use module exports for singletons, not `getInstance()` [Recommended]

An ES module is evaluated once, so an object it exports exists only once. Create process-wide objects such as connection pools and loggers once, in a module or at the program entry point. Pass in dependencies that need to be replaced or tested.

Do not put request data in module-level variables, see "Keep request context out of module-level variables" in [concurrency.md](concurrency.md).

### Use functions and object parameters for factories and builders [Recommended]

Create objects with plain functions, and `switch` inside the function when the implementation depends on a condition. When there are many optional parameters, use an object parameter with defaults, see "Use object parameters when the call site is unclear" in [idioms.md](idioms.md). Do not write `XxxFactory` classes or chained `XxxBuilder`s.

**Example:**

```ts
interface ClientOptions {
  baseUrl: string;
  timeoutMs?: number;
  retries?: number;
}

function createClient({ baseUrl, timeoutMs = 5000, retries = 0 }: ClientOptions): Client {
  return { baseUrl, timeoutMs, retries, request: (path) => send(baseUrl, path, timeoutMs, retries) };
}
```

## Behavior

### Use a map of functions for strategies, not an interface and classes [Recommended]

Write one function per case, put them in an object keyed by a union type, and check it with `satisfies`. When a member is added to the union, a missing key is a compile error. When a strategy needs several methods, make each value a plain object with those methods.

**Example:**

```ts
type ExportFormat = 'csv' | 'xlsx' | 'json';

const exporters = {
  csv: toCsv,
  xlsx: toXlsx,
  json: toJson,
} satisfies Record<ExportFormat, (rows: readonly Row[]) => Uint8Array>;

const file = exporters[format](rows);
```

**Source:** TypeScript Release Notes

### Use a discriminated union and an exhaustive `switch` for state machines [Recommended]

Each state is one member of the union and carries only the data that state needs, so illegal combinations cannot be represented. The transition function does a `switch` on `status`, and each branch returns the next state. For exhaustiveness checks, see "Handle every union branch" in [types.md](types.md).

**Example:**

```ts
type Upload =
  | { status: 'awaitingToken' }
  | { status: 'uploading'; token: string }
  | { status: 'merging'; token: string; parts: readonly string[] }
  | { status: 'done'; fileId: string };

async function advance(upload: Upload): Promise<Upload> {
  switch (upload.status) {
    case 'awaitingToken':
      return { status: 'uploading', token: await fetchToken() };
    case 'uploading':
      return { status: 'merging', token: upload.token, parts: await uploadParts(upload.token) };
    case 'merging':
      return { status: 'done', fileId: await mergeParts(upload.token, upload.parts) };
    case 'done':
      return upload;
  }
}
```

**Source:** TypeScript Handbook

### Use higher-order functions or framework middleware for cross-cutting logic [Recommended]

Write timing, retries and caching as wrapper functions that take a function and return one with the same signature. Use decorator syntax only when the framework requires it, such as NestJS or Angular.

**Example:**

```ts
function withTiming<A extends unknown[], R>(name: string, fn: (...args: A) => Promise<R>) {
  return async (...args: A): Promise<R> => {
    const start = performance.now();
    try {
      return await fn(...args);
    } finally {
      metrics.observe(name, performance.now() - start);
    }
  };
}

const loadOrder = withTiming('loadOrder', fetchOrder);
```

### Use `EventTarget` or callbacks for observers [Recommended]

With only one or two receivers, pass a callback directly. When subscription is really needed, use `EventTarget`, which both browsers and Node.js provide, or `EventEmitter` in Node.js. Do not hand-write `Subject` and `Observer` interfaces. For notifications between frontend components, use the framework's state and event mechanisms.

**Source:** MDN, Node.js API Docs

### Use generators for iteration [Recommended]

Write iteration over paged data and custom collections as `function*` or `async function*`, and have callers use `for...of` or `for await...of`. Do not write iterator classes with `hasNext()` and `next()`.

**Example:**

```ts
async function* allOrders(client: Client): AsyncGenerator<Order> {
  let cursor: string | undefined;
  do {
    const page = await client.listOrders(cursor);
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor !== undefined);
}

for await (const order of allOrders(client)) {
  await sync(order);
}
```

**Source:** MDN
