# TypeScript / JavaScript 并发

Node.js 运行时细节见 [node.md](node.md)。

## 并发模型

### 每个 Promise 都要被处理 【必须】

用 `await`、`return` 或带 `.catch()` 的调用处理。确实不等待时写 `void`，并确保函数内部已处理错误。否则错误会变成未处理拒绝，默认让进程退出。

**工具：** `@typescript-eslint/no-floating-promises`、`await-thenable`。

**来源：** typescript-eslint 文档、Node.js API 文档

### `forEach` 里不用 `async` 回调 【必须】

`forEach` 不等待回调，函数会提前返回，拒绝也没有人处理。串行用 `for...of`，并行用 `Promise.all`。

**示例：**

反例：

```ts
orders.forEach(async (order) => {
  await chargeOrder(order);
});
```

正例：

```ts
for (const order of orders) {
  await chargeOrder(order);
}
```

**工具：** `@typescript-eslint/no-misused-promises`。

### 独立操作并行执行 【推荐】

互不依赖的异步操作用 `Promise.all` 同时发起。优先 `async/await`，不混用长 `.then()` 链。

### 按失败语义选择组合方法 【推荐】

| 方法 | 行为 | 用途 |
|---|---|---|
| `Promise.all` | 任一拒绝立即拒绝 | 全部成功才有意义 |
| `Promise.allSettled` | 等全部结束，逐项给结果 | 允许部分失败 |
| `Promise.any` | 首个成功 | 多来源取其一 |
| `Promise.race` | 首个结束 | 配合取消使用 |

这些方法都不会取消其余操作。`Promise.all` 拒绝后，其他任务继续运行，需要停止时传 `AbortSignal`。

**来源：** MDN

## 取消与超时

### 外部调用都设置超时 【必须】

网络和子进程调用要有截止时间，用 `AbortSignal.timeout(ms)`。

**示例：**

```ts
const response = await fetch(url, { signal: AbortSignal.timeout(requestTimeoutMs) });
```

**来源：** MDN

### 不用 `Promise.race` 模拟超时 【推荐】

`race` 只是不再等待，底层操作继续运行，占用连接和内存。把 `AbortSignal` 传到真正执行 I/O 的 API。

**来源：** MDN

### `signal` 贯穿调用链 【推荐】

耗时函数接收 `signal` 参数并传给下层。循环中用 `signal.throwIfAborted()` 检查。外部取消和超时同时存在时用 `AbortSignal.any` 合并。`AbortSignal.timeout` 以 `TimeoutError` 中止，手动 `abort()` 默认是 `AbortError`，用 `error.name` 区分。取消是预期结果，不按故障告警。自定义 `reason` 必须是 `Error`。

**来源：** MDN

## 共享状态

### 请求上下文不放模块级变量 【必须】

模块级变量被所有请求共享，`await` 之后可能已被其他请求改写。用参数传递，或用 `AsyncLocalStorage` 传递请求 ID 这类横切信息。

**来源：** Node.js API 文档

### 跨 `await` 的检查后操作要重新确认 【必须】

判断和写入之间有 `await` 时，状态可能已变化。数据库用条件更新或唯一约束，内存状态把判断和写入放在同一段同步代码里。

**示例：**

反例：

```ts
if (!reservationBySku.has(sku)) {
  const reservation = await createReservation(sku);
  reservationBySku.set(sku, reservation);
}
```

正例：

```ts
const pending = pendingReservationBySku.get(sku) ?? createReservation(sku);
pendingReservationBySku.set(sku, pending);
const reservation = await pending.catch((error: unknown) => {
  pendingReservationBySku.delete(sku);
  throw error;
});
```

## 资源限制

### 不做无上限的并发 【必须】

**适用：** 线上服务

数量来自外部数据时，不用 `Promise.all(items.map(...))` 一次发起，否则会打满连接池和文件描述符。用 `p-limit` 这类工具或分批处理。

**示例：**

```ts
const limit = pLimit(syncConcurrency);
await Promise.all(userIds.map((userId) => limit(() => syncUser(userId))));
```

### 队列和缓存要有上限 【必须】

**适用：** 线上服务

内存队列设置长度上限，满了就拒绝或施加背压。没有上限的队列在下游变慢时会耗尽内存。流式数据见 [node.md](node.md)。

### CPU 密集任务放到 Worker 线程 【推荐】

Worker 适合 CPU 密集工作，对 I/O 帮助不大。使用固定大小的线程池，不为每个请求创建 Worker，创建开销可能超过任务本身。

**来源：** Node.js API 文档、Node.js Learn 指南

### 注意 libuv 线程池 【推荐】

`fs`、`dns.lookup`、部分 `crypto` 和 `zlib` 共用线程池，默认 4 个线程。耗时任务会拖慢其他调用。需要时用 `UV_THREADPOOL_SIZE` 调整，大任务拆分成小任务。

**来源：** Node.js API 文档、Node.js Learn 指南

## 常见竞态

### 提前创建的 Promise 要立即接管 【必须】

先创建 Promise、之后才 `await` 时，如果它在等待别的操作期间拒绝，会先触发未处理拒绝，默认让进程退出。一起放进 `Promise.all`。

**示例：**

反例：

```ts
const userPromise = fetchUser(id);
const orders = await fetchOrders(id);
const user = await userPromise;
```

正例：

```ts
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);
```

**来源：** Node.js API 文档

### 周期任务不用 `setInterval` 调异步函数 【推荐】

任务耗时超过周期时，执行会重叠。在一次完成后再安排下一次。

**示例：**

```ts
import { setTimeout as delay } from 'node:timers/promises';

async function runSyncLoop(signal: AbortSignal): Promise<void> {
  while (!signal.aborted) {
    await syncOnce(signal);
    await delay(syncIntervalMs, undefined, { signal });
  }
}
```

中止时 `delay` 以 `AbortError` 拒绝，调用方按取消处理。

### 缓存未命中时共享进行中的请求 【推荐】

并发未命中会同时打到下游。把进行中的 Promise 放进 `Map`，完成后在 `finally` 中删除，失败结果不留在缓存里。
