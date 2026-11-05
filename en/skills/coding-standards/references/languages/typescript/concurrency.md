# TypeScript / JavaScript concurrency

For Node.js runtime details, see [node.md](node.md).

## Concurrency model

### Handle every Promise [Required]

Handle it with `await`, `return`, or a call with `.catch()`. When you truly do not wait, write `void` and make sure the function handles its errors internally. Otherwise the error becomes an unhandled rejection, which exits the process by default.

**Tooling:** `@typescript-eslint/no-floating-promises`, `await-thenable`.

**Source:** typescript-eslint Docs, Node.js API Docs

### No `async` callbacks in `forEach` [Required]

`forEach` does not wait for the callback. The function returns early and nobody handles rejections. Use `for...of` for sequential work and `Promise.all` for parallel work.

**Example:**

Avoid:

```ts
orders.forEach(async (order) => {
  await chargeOrder(order);
});
```

Prefer:

```ts
for (const order of orders) {
  await chargeOrder(order);
}
```

**Tooling:** `@typescript-eslint/no-misused-promises`.

### Run independent operations in parallel [Recommended]

Start async operations that do not depend on each other together with `Promise.all`. Prefer `async/await` and do not mix in long `.then()` chains.

### Choose the combinator by failure semantics [Recommended]

| Method | Behavior | Use |
|---|---|---|
| `Promise.all` | Rejects immediately when any rejects | Only all succeeding is meaningful |
| `Promise.allSettled` | Waits for all to finish, gives a result per item | Partial failure is allowed |
| `Promise.any` | First success | Take one of several sources |
| `Promise.race` | First to finish | Use with cancellation |

None of these methods cancel the remaining operations. After `Promise.all` rejects, the other tasks keep running. Pass an `AbortSignal` when they need to stop.

**Source:** MDN

## Cancellation and timeouts

### Set a timeout on all external calls [Required]

Network and subprocess calls need a deadline. Use `AbortSignal.timeout(ms)`.

**Example:**

```ts
const response = await fetch(url, { signal: AbortSignal.timeout(requestTimeoutMs) });
```

**Source:** MDN

### Do not simulate timeouts with `Promise.race` [Recommended]

`race` only stops waiting. The underlying operation keeps running and holds connections and memory. Pass an `AbortSignal` to the API that actually performs the I/O.

**Source:** MDN

### Pass `signal` through the call chain [Recommended]

Long-running functions accept a `signal` parameter and pass it down. In loops, check with `signal.throwIfAborted()`. When external cancellation and a timeout both exist, combine them with `AbortSignal.any`. `AbortSignal.timeout` aborts with `TimeoutError`, and a manual `abort()` defaults to `AbortError`. Tell them apart with `error.name`. Cancellation is an expected outcome, so do not alert on it as a failure. A custom `reason` must be an `Error`.

**Source:** MDN

## Shared state

### Keep request context out of module-level variables [Required]

Module-level variables are shared by all requests, and after an `await` another request may have overwritten them. Pass values as parameters, or use `AsyncLocalStorage` for cross-cutting information such as request IDs.

**Source:** Node.js API Docs

### Recheck before acting across `await` [Required]

When there is an `await` between the check and the write, the state may have changed. For databases, use conditional updates or unique constraints. For in-memory state, put the check and the write in the same synchronous block.

**Example:**

Avoid:

```ts
if (!reservationBySku.has(sku)) {
  const reservation = await createReservation(sku);
  reservationBySku.set(sku, reservation);
}
```

Prefer:

```ts
const pending = pendingReservationBySku.get(sku) ?? createReservation(sku);
pendingReservationBySku.set(sku, pending);
const reservation = await pending.catch((error: unknown) => {
  pendingReservationBySku.delete(sku);
  throw error;
});
```

## Resource limits

### No unbounded concurrency [Required]

**Applies to:** production service

When the count comes from external data, do not start everything at once with `Promise.all(items.map(...))`, or you will exhaust the connection pool and file descriptors. Use a tool such as `p-limit`, or process in batches.

**Example:**

```ts
const limit = pLimit(syncConcurrency);
await Promise.all(userIds.map((userId) => limit(() => syncUser(userId))));
```

### Bound queues and caches [Required]

**Applies to:** production service

Set a length limit on in-memory queues. When one is full, reject or apply backpressure. An unbounded queue exhausts memory when the downstream slows down. For streaming data, see [node.md](node.md).

### Move CPU-intensive tasks to Worker threads [Recommended]

Workers suit CPU-intensive work and do little for I/O. Use a fixed-size thread pool. Do not create a Worker per request: the creation cost can exceed the task itself.

**Source:** Node.js API Docs, Node.js Learn Guides

### Watch the libuv thread pool [Recommended]

`fs`, `dns.lookup`, and parts of `crypto` and `zlib` share a thread pool with 4 threads by default. Long-running tasks slow down other calls. Adjust it with `UV_THREADPOOL_SIZE` when needed, and split large tasks into small ones.

**Source:** Node.js API Docs, Node.js Learn Guides

## Common races

### Take over early-created Promises immediately [Required]

If you create a Promise first and `await` it later, and it rejects while you are waiting on another operation, an unhandled rejection fires first, which exits the process by default. Put them together in `Promise.all`.

**Example:**

Avoid:

```ts
const userPromise = fetchUser(id);
const orders = await fetchOrders(id);
const user = await userPromise;
```

Prefer:

```ts
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);
```

**Source:** Node.js API Docs

### Do not call async functions from `setInterval` for periodic tasks [Recommended]

When a task takes longer than the interval, runs overlap. Schedule the next run after the current one completes.

**Example:**

```ts
import { setTimeout as delay } from 'node:timers/promises';

async function runSyncLoop(signal: AbortSignal): Promise<void> {
  while (!signal.aborted) {
    await syncOnce(signal);
    await delay(syncIntervalMs, undefined, { signal });
  }
}
```

On abort, `delay` rejects with `AbortError`. The caller treats it as cancellation.

### Share in-flight requests on a cache miss [Recommended]

Concurrent misses all hit the downstream at once. Put the in-flight Promise in a `Map`, delete it in `finally` when it completes, and keep failed results out of the cache.
