# Node.js 运行时

并发写法见 [concurrency.md](concurrency.md)。接口和架构规范见后端层。

## 事件循环

### 不阻塞事件循环 【必须】

**适用：** 线上服务

所有请求共用一个事件循环，一个回调运行过久，所有客户端都要等待。`await` 之后的同步代码同样占用它。回调的耗时要和输入规模解耦，或给输入设置上限。

**来源：** Node.js Learn 指南

### 请求路径上不用同步 API 【必须】

**适用：** 线上服务

`fs` 的 `*Sync`、`crypto.pbkdf2Sync`、`zlib.inflateSync`、`execSync` 只用于启动阶段和脚本。

**工具：** 开发环境用 `--trace-sync-io` 定位。

**来源：** Node.js Learn 指南、Express 文档

### 计算任务分片或卸载 【推荐】

简单循环可以分片，每批之间用 `setImmediate` 让出。复杂计算放到 Worker 线程池。大 JSON 的解析和序列化也算计算任务。

**示例：**

```ts
import { setImmediate as yieldToEventLoop } from 'node:timers/promises';

for (const [index, row] of rows.entries()) {
  transformRow(row);
  if (index % yieldEveryRows === 0) {
    await yieldToEventLoop();
  }
}
```

**来源：** Node.js Learn 指南

### 不用微任务让出 【推荐】

`process.nextTick` 和 Promise 回调在进入下一阶段前全部执行，递归调度会饿死 I/O。需要让 I/O 运行时用 `setImmediate`。

**来源：** Node.js Learn 指南

### 延迟到当前操作之后用 `queueMicrotask` 【推荐】

`process.nextTick` 是遗留 API，用 `queueMicrotask`。不要依赖两者的相对顺序，它在 CommonJS 和 ESM 中不同。

**来源：** Node.js API 文档

### 监控事件循环延迟 【推荐】

**适用：** 线上服务

生产环境采集 `perf_hooks.monitorEventLoopDelay()` 和 `performance.eventLoopUtilization()`，在延迟升高时告警。

**来源：** Node.js API 文档

## 进程信号

### 处理 `SIGTERM` 并优雅退出 【必须】

**适用：** 线上服务

收到信号后停止接收新请求，等待进行中的请求，关闭连接池，再退出。安装监听器后默认退出行为被移除，处理函数必须自己结束进程。`SIGKILL` 无法监听，所以要设置退出期限，并且期限短于编排系统的宽限期。

**示例：**

```ts
async function shutdown(signal: NodeJS.Signals): Promise<void> {
  logger.info({ signal }, '开始退出');
  setTimeout(() => process.exit(1), shutdownTimeoutMs).unref();
  try {
    await new Promise((resolve) => server.close(resolve));
    await pool.end();
  } catch (error) {
    logger.error({ err: error }, '退出失败');
    process.exitCode = 1;
  }
}

process.once('SIGTERM', (signal) => void shutdown(signal));
process.once('SIGINT', (signal) => void shutdown(signal));
```

`server.close` 关闭空闲连接后，等待活动请求结束。

**来源：** Node.js API 文档、Express 文档

### 用 `process.exitCode`，少用 `process.exit()` 【推荐】

`process.exit()` 立即结束，待写入的 stdout 和 stderr 可能丢失。设置 `process.exitCode` 后让进程自然结束。`'exit'` 监听器里只能做同步操作。

**来源：** Node.js API 文档

### 未捕获异常记录后退出 【必须】

**适用：** 线上服务

进程已处于未定义状态，不能继续服务。只做同步清理和记录，由进程管理器重启。默认模式下未处理拒绝会转成 `uncaughtException`，监听后通过 `origin` 区分。不写 `process.on('unhandledRejection', () => {})` 这样的吞错处理。

**示例：**

```ts
process.on('uncaughtException', (error, origin) => {
  logger.fatal({ err: error, origin }, '未捕获异常');
  process.exit(1);
});
```

**来源：** Node.js API 文档、Fastify 文档

## Express/Fastify/NestJS 结构

### 按框架组织功能目录 【推荐】

每个功能一个目录，框架入口、业务代码和类型放在一起。

Express：

```text
src/
  app.ts
  server.ts
  orders/
    order-routes.ts
    order-service.ts
    order-types.ts
```

Fastify：

```text
src/
  app.ts
  server.ts
  orders/
    index.ts
    order-service.ts
    order-schemas.ts
```

`index.ts` 导出这个功能的插件。

NestJS：

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

Express 和 Fastify 把 `app.ts` 和 `server.ts` 分开，`app.ts` 只构建应用，测试时不用监听端口。

### 框架代码止于路由层 【推荐】

路由层只解析请求、调用服务和构造响应。业务代码不引用 `req`、`reply` 这类框架对象，这样可以脱离 HTTP 测试。

### Express：应用构建和中间件顺序明确 【推荐】

按功能拆 `Router`，错误中间件最后注册，生产环境设置 `NODE_ENV=production`。Express 5 自动处理 `async` 处理函数的拒绝，不需要手写包装函数。请求校验放在路由入口。

**来源：** Express 文档

### Fastify：用插件划分封装上下文 【推荐】

每个功能是一个插件，`register` 创建封装上下文，装饰器和钩子只对子上下文可见。需要共享给父级时用 `fastify-plugin`。路由写 JSON schema 校验输入、序列化输出。`async` 处理函数直接返回响应，调用 `reply.send` 时必须 `return reply`，否则存在竞态。

**来源：** Fastify 文档

### NestJS：按功能模块组织 【推荐】

模块默认封装 provider，只 `exports` 需要给外部用的部分。provider 默认单例并跨请求共享，不在实例字段保存请求状态。`REQUEST` 作用域会沿注入链向上传递，确有必要时才用。

**来源：** NestJS 文档

### NestJS：避免循环依赖 【推荐】

先调整模块边界，不用 `forwardRef` 掩盖问题。模块和 provider 类不通过 barrel 文件导入，它是循环依赖的常见来源。

**来源：** NestJS 文档

### NestJS：全局校验和退出钩子 【必须】

全局 `ValidationPipe` 开启 `whitelist` 和 `forbidNonWhitelisted`，拒绝未声明字段。调用 `app.enableShutdownHooks()`，否则信号不会触发 `onApplicationShutdown`。

**来源：** NestJS 文档

## Stream

### 大数据用流处理 【必须】

**适用：** 线上服务

文件、上传和导出不整体读入内存。`readFile` 一个大文件时，内存和耗时都随文件大小增长。

**来源：** Node.js Learn 指南

### 用 `pipeline`，不用 `.pipe()` 【必须】

`.pipe()` 不传播错误，也不销毁其他流，会泄漏文件描述符。`node:stream/promises` 的 `pipeline` 传播错误、统一清理，并支持 `signal`。

**示例：**

反例：

```ts
createReadStream(source).pipe(createGzip()).pipe(createWriteStream(target));
```

正例：

```ts
await pipeline(createReadStream(source), createGzip(), createWriteStream(target), { signal });
```

**来源：** Node.js API 文档、Node.js Learn 指南

### 手动写入时遵守背压 【必须】

**适用：** 线上服务

`write()` 返回 `false` 后停止写入，等 `'drain'` 再继续，否则数据堆积在内存里。优先用 `pipeline` 或异步生成器，由它们处理背压。

**示例：**

```ts
if (!output.write(chunk)) {
  await once(output, 'drain');
}
```

**来源：** Node.js Learn 指南、Node.js API 文档

### 流的 `'error'` 必须有去向 【必须】

没有监听器的 `'error'` 事件会让进程退出，网络 socket 也一样。使用 `pipeline` 或显式监听。向 HTTP 响应写入时注意 `pipeline` 失败后 socket 已被销毁，不能再发错误响应。

**来源：** Node.js API 文档、Node.js 安全最佳实践

## 内存泄漏

### 无界结构必须有上限 【必须】

**适用：** 线上服务

Node.js 进程长期运行，模块级 `Map`、数组和缓存只增不减就是泄漏。设置容量和过期时间，比如 LRU。键是对象且生命周期跟随对象时用 `WeakMap`。

**来源：** Node.js Learn 指南

### 监听器和定时器成对清理 【必须】

**适用：** 线上服务

长生命周期对象上的监听器在资源结束时移除，`setInterval` 要 `clearInterval`。`MaxListenersExceededWarning` 通常代表泄漏，不要用 `setMaxListeners(0)` 压掉。长期存在的 `AbortSignal` 上的监听器也要移除。

**示例：**

```ts
function subscribe(session: Session): void {
  const onPrice = (price: string): void => session.updatePrice(price);
  bus.on('price', onPrice);
  session.once('close', () => bus.off('price', onPrice));
}
```

**来源：** Node.js API 文档

### 闭包不持有大对象 【推荐】

长期存在的回调只捕获需要的字段，不捕获整个请求或响应。未结束的 Promise 也会持有其闭包，反复把长期挂起的 Promise 传给 `Promise.all` 或 `race` 会累积处理函数。

**来源：** MDN

### 堆上限和容器内存匹配 【推荐】

**适用：** 线上服务

用 `--max-old-space-size` 或 `--max-old-space-size-percentage` 设置，并为堆外内存留余量。调大堆上限不能修复泄漏。

**来源：** Node.js Learn 指南、Node.js API 文档
