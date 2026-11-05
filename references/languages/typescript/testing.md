# TypeScript / JavaScript 测试

测试策略见 [general/testing-strategy.md](../../general/testing-strategy.md)。本文只写工具用法。Vitest 运行时不检查类型，类型检查的要求见 [project.md](project.md)。

## 框架

### 默认使用 Vitest 【推荐】

新项目用 Vitest，它直接支持 TypeScript 和 ESM。已使用 Jest 的项目沿用，一个仓库不混用两种框架。无依赖的小工具可以用内置的 `node:test`。

**来源：** Vitest 文档、NestJS 文档

### 类型约束用类型测试 【推荐】

公共泛型和工具类型写在 `*.test-d.ts`，用 `expectTypeOf` 断言，通过 `vitest --typecheck` 运行。预期报错的位置用 `@ts-expect-error`。

**来源：** Vitest 文档

## 组织方式

### 测试文件紧邻源码 【推荐】

`order-service.ts` 的测试放在同目录的 `order-service.test.ts`。集成测试放在 `test/` 或用独立的 Vitest project，让单元测试可以单独运行。

**来源：** Vitest 文档

### 模块级状态在 beforeEach 中重置 【推荐】

测试独立性的要求见 [testing-strategy.md](../../general/testing-strategy.md)。每个测试自行准备数据，模块级可变状态在 `beforeEach` 中重置。打开 `--no-isolate` 提速的前提是测试没有副作用残留。

**来源：** Vitest 文档

### 异步断言必须等待 【必须】

`expect(promise).rejects` 要 `await`，否则断言失败不会让测试失败。不用 `try/catch` 加 `fail()` 断言异常。

**示例：**

反例：

```ts
test('订单不存在时报错', () => {
  expect(loadOrder('missing')).rejects.toThrow(OrderNotFoundError);
});
```

正例：

```ts
test('订单不存在时报错', async () => {
  await expect(loadOrder('missing')).rejects.toThrow(OrderNotFoundError);
});
```

**工具：** `@typescript-eslint/no-floating-promises` 覆盖测试文件。

**来源：** Google TypeScript 风格指南

### 不提交 `.only` 【必须】

**适用：** 多人协作

`.only` 会让其余测试被跳过。Vitest 在 CI 环境默认禁止 `.only`，不要打开 `allowOnly`。

**来源：** Vitest 文档

### 并发测试使用上下文的 `expect` 【推荐】

`test.concurrent` 中快照和断言用测试上下文提供的 `expect`，否则结果可能归到错误的测试上。

**来源：** Vitest 文档

## 参数化

### 同逻辑多输入用 `test.each` 或 `test.for` 【推荐】

用例名写出输入和预期。`test.each` 展开数组参数，`test.for` 不展开，并且能拿到测试上下文。

**示例：**

```ts
test.each([
  { totalCents: 0, expected: 'free' },
  { totalCents: 4999, expected: 'standard' },
  { totalCents: 5000, expected: 'free' },
])('$totalCents 分对应 $expected 运费', ({ totalCents, expected }) => {
  expect(selectShipping(totalCents)).toBe(expected);
});
```

**来源：** Vitest 文档

## 替身

### 优先注入依赖，再考虑模块替身 【推荐】

能通过参数传入替身时，不用 `vi.mock`。`vi.mock` 被提升到文件顶部，比所有导入先执行，依赖模块路径，重构时容易失效。

**来源：** Vitest 文档

### 替身状态每个测试还原 【推荐】

配置 `restoreMocks: true`，或在 `afterEach` 调用 `vi.restoreAllMocks()`。`vi.spyOn` 不还原时会污染后续测试。

**例外：** `restoreMocks` 可能影响正在运行的并发测试，这类文件手动还原。

**来源：** Vitest 文档

### 时间用假定时器控制 【推荐】

不写真实 `sleep`。用 `vi.useFakeTimers()` 和 `vi.setSystemTime()`，它也会替换 `Date`。`afterEach` 中调用 `vi.useRealTimers()`。

**示例：**

```ts
beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date('2026-01-01T00:00:00Z'));
});

afterEach(() => {
  vi.useRealTimers();
});
```

**来源：** Vitest 文档

### HTTP 依赖用 MSW 拦截 【推荐】

不替换 `fetch` 本身。MSW 在网络层拦截，应用代码不用修改。未声明的请求配置为报错，避免测试访问真实网络。

**来源：** Vitest 文档

### 替身的类型要完整 【推荐】

用 `vi.fn<typeof sendMail>()`、`vi.mocked()` 保留类型。签名变化时编译器能发现替身过期。必须用 `any` 时只限测试文件内局部使用。

**来源：** Google TypeScript 风格指南

## 集成测试

### HTTP 接口不开真实端口 【推荐】

Fastify 用 `app.inject()`，Express 和 NestJS 用 Supertest。前提是应用实例和 `listen` 分开。

**示例：**

```ts
const response = await app.inject({ method: 'GET', url: '/orders/1' });

expect(response.statusCode).toBe(200);
```

**来源：** Fastify 文档、NestJS 文档

### 数据库用真实实例 【推荐】

SQL 和事务行为用容器中的真实数据库验证，不用内存替代品。每个测试用事务回滚或独立 schema 隔离。

### 测试结束释放资源 【推荐】

`afterAll` 中关闭应用、连接池和定时器。进程退不出来时用 `process.getActiveResourcesInfo()` 排查，不用强制退出掩盖。

**来源：** Node.js API 文档

## 基准测试

### 基准测试用 Vitest `bench` 【推荐】

文件命名为 `*.bench.ts`。结果只在同一环境内比较，开发机和 CI 不能互比。

**来源：** Vitest 文档

### 基准函数要使用结果 【推荐】

没有可观察副作用的代码可能被引擎优化掉，得到错误结论。返回或消费计算结果。

**来源：** Vitest 文档

### 不把基准结果做成硬性门禁 【推荐】

CPU 负载和 GC 都会影响结果。需要断言时设置相对差异容忍度。

**来源：** Vitest 文档
