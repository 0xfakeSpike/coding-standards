# TypeScript 设计模式写法

本文写设计模式在 TypeScript 里的惯用形态。什么时候用哪个模式见 [design-patterns.md](../../general/design-patterns.md)。总的原则是能用函数和普通对象表达的，不写类，见 [types.md](types.md) 的“纯数据用接口，不用类”和 [idioms.md](idioms.md) 的“不写只转调一次的函数和无意义的封装”。

## 结构

### 只有一个实现时不定义接口 【推荐】

不写 `IUserService` 加 `UserServiceImpl`。类型是结构化的，测试替身只要形状相同就能传入，不需要实现方预先声明接口。有多个实现，或者使用方只用到其中几个方法时，由使用方声明它需要的形状。

### 字段直接暴露，不写 getter 和 setter 【推荐】

不会修改的字段用 `readonly`。只有需要计算、校验或保持不变量时，才用 `get`、`set` 访问器或方法。

### 单例用模块导出，不写 `getInstance()` 【推荐】

ES 模块只求值一次，模块导出的对象天然只有一份。连接池、logger 这类随进程存在的对象在模块或程序入口创建一次。需要替换或测试的依赖从参数传入。

请求相关的数据不放模块级变量，见 [concurrency.md](concurrency.md) 的“请求上下文不放模块级变量”。

### 工厂和建造者用函数和对象参数 【推荐】

创建对象用普通函数，需要按条件选择实现时在函数里 `switch`。可选参数多时用对象参数加默认值，见 [idioms.md](idioms.md) 的“调用处读不懂时用对象参数”。不写 `XxxFactory` 类和链式的 `XxxBuilder`。

**示例：**

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

## 行为

### 策略用函数映射，不写接口加类 【推荐】

每种情况一个函数，放进以联合类型为键的对象，并用 `satisfies` 检查。新增联合成员时，漏写的键会编译报错。策略需要多个方法时，值换成包含这些方法的普通对象。

**示例：**

```ts
type ExportFormat = 'csv' | 'xlsx' | 'json';

const exporters = {
  csv: toCsv,
  xlsx: toXlsx,
  json: toJson,
} satisfies Record<ExportFormat, (rows: readonly Row[]) => Uint8Array>;

const file = exporters[format](rows);
```

**来源：** TypeScript 发布说明

### 状态机用可辨识联合加穷尽 `switch` 【推荐】

每个状态是联合的一个成员，只带这个状态需要的数据，非法组合无法表示。迁移函数按 `status` 做 `switch`，每个分支返回下一个状态。穷尽检查见 [types.md](types.md) 的“联合类型的分支要穷尽”。

**示例：**

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

**来源：** TypeScript 手册

### 横切逻辑用高阶函数或框架中间件 【推荐】

计时、重试、缓存这类逻辑写成接收函数、返回同签名函数的包装函数。装饰器语法只在框架要求时使用，比如 NestJS、Angular。

**示例：**

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

### 观察者用 `EventTarget` 或回调 【推荐】

只有一两个接收方时直接传回调。确实需要订阅时，用浏览器和 Node.js 都有的 `EventTarget`，Node.js 中也可以用 `EventEmitter`。不手写 `Subject`、`Observer` 接口。前端组件之间的通知用框架的状态和事件机制。

**来源：** MDN、Node.js API 文档

### 遍历用生成器 【推荐】

分页数据和自定义集合的遍历写成 `function*` 或 `async function*`，调用方用 `for...of` 或 `for await...of`。不写带 `hasNext()`、`next()` 的迭代器类。

**示例：**

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

**来源：** MDN
