# Node.js runtime

For concurrency patterns, see [concurrency.md](concurrency.md). For API and architecture rules, see the backend layer.

## Event loop

### Do not block the event loop [Required]

**Applies to:** production service

All requests share one event loop. When one callback runs too long, every client waits. Synchronous code after an `await` occupies it too. Decouple callback duration from input size, or cap the input.

**Source:** Node.js Learn Guides

### No synchronous APIs on the request path [Required]

**Applies to:** production service

Use `*Sync` methods of `fs`, `crypto.pbkdf2Sync`, `zlib.inflateSync`, and `execSync` only during startup and in scripts.

**Tooling:** Use `--trace-sync-io` in development to locate them.

**Source:** Node.js Learn Guides, Express Docs

### Partition or offload computation [Recommended]

Simple loops can be split into batches, yielding with `setImmediate` between batches. Put complex computation in a Worker thread pool. Parsing and serializing large JSON also counts as computation.

**Example:**

```ts
import { setImmediate as yieldToEventLoop } from 'node:timers/promises';

for (const [index, row] of rows.entries()) {
  transformRow(row);
  if (index % yieldEveryRows === 0) {
    await yieldToEventLoop();
  }
}
```

**Source:** Node.js Learn Guides

### Do not yield with microtasks [Recommended]

`process.nextTick` and Promise callbacks all run before the next phase, so recursive scheduling starves I/O. Use `setImmediate` when I/O needs to run.

**Source:** Node.js Learn Guides

### Use `queueMicrotask` to defer until after the current operation [Recommended]

`process.nextTick` is a legacy API. Use `queueMicrotask`. Do not rely on the relative order of the two: it differs between CommonJS and ESM.

**Source:** Node.js API Docs

### Monitor event loop delay [Recommended]

**Applies to:** production service

Collect `perf_hooks.monitorEventLoopDelay()` and `performance.eventLoopUtilization()` in production, and alert when delay rises.

**Source:** Node.js API Docs

## Process signals

### Handle `SIGTERM` and exit gracefully [Required]

**Applies to:** production service

On a signal, stop accepting new requests, wait for in-flight requests, close connection pools, then exit. Installing a listener removes the default exit behavior, so the handler must end the process itself. `SIGKILL` cannot be listened for, so set an exit deadline shorter than the orchestrator's grace period.

**Example:**

```ts
async function shutdown(signal: NodeJS.Signals): Promise<void> {
  logger.info({ signal }, 'Shutting down');
  setTimeout(() => process.exit(1), shutdownTimeoutMs).unref();
  try {
    await new Promise((resolve) => server.close(resolve));
    await pool.end();
  } catch (error) {
    logger.error({ err: error }, 'Shutdown failed');
    process.exitCode = 1;
  }
}

process.once('SIGTERM', (signal) => void shutdown(signal));
process.once('SIGINT', (signal) => void shutdown(signal));
```

`server.close` closes idle connections, then waits for active requests to finish.

**Source:** Node.js API Docs, Express Docs

### Use `process.exitCode`, and use `process.exit()` sparingly [Recommended]

`process.exit()` ends immediately, and pending stdout and stderr writes may be lost. Set `process.exitCode` and let the process end naturally. Do only synchronous work in `'exit'` listeners.

**Source:** Node.js API Docs

### Log uncaught exceptions, then exit [Required]

**Applies to:** production service

The process is in an undefined state and cannot keep serving. Do only synchronous cleanup and logging, and let the process manager restart it. In default mode, unhandled rejections turn into `uncaughtException`. When listening, tell them apart by `origin`. Do not write error-swallowing handlers such as `process.on('unhandledRejection', () => {})`.

**Example:**

```ts
process.on('uncaughtException', (error, origin) => {
  logger.fatal({ err: error, origin }, 'Uncaught exception');
  process.exit(1);
});
```

**Source:** Node.js API Docs, Fastify Docs

## Express/Fastify/NestJS structure

### Organize feature directories by framework [Recommended]

Use one directory per feature, with the framework entry point, business code, and types together.

Express:

```text
src/
  app.ts
  server.ts
  orders/
    order-routes.ts
    order-service.ts
    order-types.ts
```

Fastify:

```text
src/
  app.ts
  server.ts
  orders/
    index.ts
    order-service.ts
    order-schemas.ts
```

`index.ts` exports the plugin for this feature.

NestJS:

```text
src/
  main.ts
  app.module.ts
  orders/
    orders.module.ts
    orders.controller.ts
    orders.service.ts
    dto/
      create-order.dto.ts
```

Express and Fastify keep `app.ts` and `server.ts` separate. `app.ts` only builds the app, so tests do not need to listen on a port.

### Keep framework code in the route layer [Recommended]

The route layer only parses requests, calls services, and builds responses. Business code does not reference framework objects such as `req` and `reply`, so it can be tested without HTTP.

### Express: explicit app construction and middleware order [Recommended]

Split `Router`s by feature, register error middleware last, and set `NODE_ENV=production` in production. Express 5 handles rejections from `async` handlers automatically, so no hand-written wrapper function is needed. Put request validation at the route entry.

**Source:** Express Docs

### Fastify: use plugins to define encapsulation contexts [Recommended]

Each feature is a plugin. `register` creates an encapsulation context, and decorators and hooks are visible only to child contexts. Use `fastify-plugin` when they need to be shared with the parent. Routes declare JSON schemas to validate input and serialize output. `async` handlers return the response directly. When calling `reply.send`, you must `return reply`, or there is a race condition.

**Source:** Fastify Docs

### NestJS: organize by feature module [Recommended]

Modules encapsulate providers by default. Put in `exports` only what outside code needs. Providers are singletons by default and shared across requests, so do not keep request state in instance fields. `REQUEST` scope propagates up the injection chain. Use it only when truly necessary.

**Source:** NestJS Docs

### NestJS: avoid circular dependencies [Recommended]

Adjust module boundaries first. Do not hide the problem with `forwardRef`. Do not import modules and provider classes through barrel files, which are a common source of circular dependencies.

**Source:** NestJS Docs

### NestJS: global validation and shutdown hooks [Required]

Enable `whitelist` and `forbidNonWhitelisted` on the global `ValidationPipe` to reject undeclared fields. Call `app.enableShutdownHooks()`, or signals will not trigger `onApplicationShutdown`.

**Source:** NestJS Docs

## Streams

### Use streams for large data [Required]

**Applies to:** production service

Do not read files, uploads, and exports entirely into memory. When you `readFile` a large file, memory and time both grow with the file size.

**Source:** Node.js Learn Guides

### Use `pipeline`, not `.pipe()` [Required]

`.pipe()` does not propagate errors or destroy the other streams, which leaks file descriptors. `pipeline` from `node:stream/promises` propagates errors, cleans up uniformly, and supports `signal`.

**Example:**

Avoid:

```ts
createReadStream(source).pipe(createGzip()).pipe(createWriteStream(target));
```

Prefer:

```ts
await pipeline(createReadStream(source), createGzip(), createWriteStream(target), { signal });
```

**Source:** Node.js API Docs, Node.js Learn Guides

### Respect backpressure on manual writes [Required]

**Applies to:** production service

After `write()` returns `false`, stop writing and wait for `'drain'` before continuing, or data piles up in memory. Prefer `pipeline` or async generators, which handle backpressure for you.

**Example:**

```ts
if (!output.write(chunk)) {
  await once(output, 'drain');
}
```

**Source:** Node.js Learn Guides, Node.js API Docs

### Every stream `'error'` must have a destination [Required]

An `'error'` event with no listener exits the process, and network sockets are no exception. Use `pipeline` or listen explicitly. When writing to an HTTP response, note that after `pipeline` fails the socket is already destroyed, so you cannot send an error response.

**Source:** Node.js API Docs, Node.js Security Best Practices

## Memory leaks

### Cap unbounded structures [Required]

**Applies to:** production service

Node.js processes run for a long time, so module-level `Map`s, arrays, and caches that only grow are leaks. Set a capacity and expiry, for example with an LRU. When keys are objects and the lifetime follows the object, use `WeakMap`.

**Source:** Node.js Learn Guides

### Clean up listeners and timers in pairs [Required]

**Applies to:** production service

Remove listeners on long-lived objects when the resource ends, and pair `setInterval` with `clearInterval`. `MaxListenersExceededWarning` usually signals a leak. Do not silence it with `setMaxListeners(0)`. Also remove listeners on long-lived `AbortSignal`s.

**Example:**

```ts
function subscribe(session: Session): void {
  const onPrice = (price: string): void => session.updatePrice(price);
  bus.on('price', onPrice);
  session.once('close', () => bus.off('price', onPrice));
}
```

**Source:** Node.js API Docs

### Keep large objects out of closures [Recommended]

Long-lived callbacks capture only the fields they need, not the whole request or response. An unsettled Promise also retains its closures, and repeatedly passing a long-pending Promise to `Promise.all` or `race` accumulates handlers.

**Source:** MDN

### Match the heap limit to container memory [Recommended]

**Applies to:** production service

Set it with `--max-old-space-size` or `--max-old-space-size-percentage`, and leave headroom for off-heap memory. Raising the heap limit does not fix leaks.

**Source:** Node.js Learn Guides, Node.js API Docs
