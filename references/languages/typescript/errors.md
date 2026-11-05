# TypeScript / JavaScript 错误处理

语言无关的原则见 [general/error-handling.md](../../general/error-handling.md)。本文只写语言机制。

## 错误机制

### 只抛出 `Error` 及其子类 【推荐】

只有 `Error` 才有堆栈。这条同样适用于 `Promise.reject` 和 `AbortController.abort(reason)`。总是用 `new Error()`。

**示例：**

反例：

```ts
throw 'order not found';
await Promise.reject({ code: 404 });
```

正例：

```ts
throw new OrderNotFoundError(orderId);
```

**例外：** 框架用抛出做控制流时按框架写法，比如 `throw redirect()`。捕获到这类值的 `catch` 要原样重新抛出。

**工具：** `@typescript-eslint/only-throw-error`、`prefer-promise-reject-errors`。

**来源：** Google TypeScript 风格指南、typescript-eslint 文档

### 捕获到的值按 `unknown` 处理 【推荐】

`strict` 包含 `useUnknownInCatchVariables`，`catch` 变量是 `unknown`。`.catch()` 回调的参数也写成 `unknown`。读取属性前先用 `instanceof` 判断。

**工具：** `@typescript-eslint/use-unknown-in-catch-callback-variable`。

**来源：** TSConfig 参考、typescript-eslint 文档

### 预期内的失败用返回值，其余用异常 【推荐】

调用方必须处理的业务失败可以用可辨识联合返回，类型系统会强制处理。程序缺陷、依赖故障和无法在本层决策的失败抛异常。同一模块内用同一种方式。

### 不写空的 `catch` 【必须】

捕获后什么都不做会吞掉错误。确实可以忽略时，只捕获那一种错误，其余重新抛出，原因就近写一句注释。

**工具：** ESLint `no-empty`。

**来源：** Google TypeScript 风格指南

### `try` 只包住可能抛错的代码 【推荐】

`try` 块过大时看不出哪一行会失败，也可能捕获到无关错误。

**来源：** Google TypeScript 风格指南

### 需要捕获时用 `return await` 【必须】

`try` 中 `return promise` 不等待，拒绝不会进入当前 `catch`，`finally` 也会提前执行。

**示例：**

```ts
try {
  return await loadOrder(id);
} catch (error) {
  throw new OrderLoadError(id, { cause: error });
}
```

**工具：** `@typescript-eslint/return-await`。

**来源：** typescript-eslint 文档

### 资源释放用 `finally` 或 `using` 【必须】

出错路径也要释放资源。`using` 和 `await using` 需要运行时支持和 `Symbol.dispose`，不支持时用 `finally`。

**来源：** MDN

## 自定义错误及其归属

### 自定义错误继承 `Error` 并设置 `name` 【推荐】

调用方需要按类型分支时才定义子类。构造函数接收 `ErrorOptions` 以支持 `cause`，业务字段写成只读属性。

**示例：**

```ts
export class OrderLoadError extends Error {
  override readonly name = 'OrderLoadError';
  readonly orderId: string;

  constructor(orderId: string, options?: ErrorOptions) {
    super(`加载订单失败 orderId=${orderId}`, options);
    this.orderId = orderId;
  }
}
```

**来源：** Google TypeScript 风格指南

### 错误类定义在使用它的模块内 【必须】

不建全局错误表。跨包判断时用 `code` 字段，同一个包被加载两份时 `instanceof` 会失败。

**来源：** 团队

## 包装与判断

### 重新抛出时用 `cause` 保留根因 【必须】

消息写清正在做什么、处理的是哪个对象。`cause` 放在第二个参数的 `options` 对象里。`Error` 可以赋给 `ErrorOptions` 类型，把错误对象直接作为第二个参数不会报错，但会丢掉根因。

**示例：**

反例：

```ts
throw new Error('扣款失败', error);
```

正例：

```ts
throw new PaymentError(`扣款失败 orderId=${order.id}`, { cause: error });
```

**来源：** Google TypeScript 风格指南、MDN

### 用 `instanceof` 或 `code` 判断，不解析消息 【必须】

错误消息不是稳定接口。Node.js 内置错误用 `error.code`，比如 `ENOENT`。

**来源：** Node.js API 文档

### 批量失败用 `AggregateError` 或逐项结果 【推荐】

`Promise.allSettled` 的结果逐项检查，不能只 `await` 后认为全部成功。失败要向上暴露时用 `AggregateError` 汇总。

**来源：** MDN

## 日志配合

### 把错误对象交给日志，不只记 `message` 【推荐】

只记 `message` 会丢失堆栈和 `cause`。使用结构化日志，比如 Pino，错误放在专门字段。

**示例：**

```ts
logger.error({ err: error, orderId }, '扣款失败');
```

**来源：** Express 文档

### 中间层只包装，不记录 【推荐】

原则见 [general/error-handling.md](../../general/error-handling.md)。中间层用 `cause` 包装后抛出，在框架错误处理器或任务入口统一记录。

## 框架边界

### 错误在框架错误处理器统一转换 【推荐】

**适用：** 线上服务

路由处理函数只抛出业务错误，HTTP 状态码和响应体的映射放在一处。未识别的错误返回通用 500，堆栈不进入响应。

| 框架 | 位置 | 注意 |
|---|---|---|
| Express 5 | 四参数错误中间件，最后注册 | `async` 处理函数的拒绝自动进入 `next(err)`，已开始响应时交给 `next(err)` |
| Fastify | `setErrorHandler` | 按封装上下文生效 |
| NestJS | 异常过滤器 | 非 `HttpException` 默认返回 500 |

**来源：** Express 文档、Fastify 文档、NestJS 文档

### 回调和事件中的错误要接住 【必须】

`try` 接不住之后才执行的回调。`EventEmitter` 的 `'error'` 没有监听器时会抛出并结束进程。`async` 函数不要直接作为事件监听器或 `void` 回调，它的拒绝没有人处理。

**工具：** `@typescript-eslint/no-misused-promises`。

**来源：** Node.js API 文档、typescript-eslint 文档
