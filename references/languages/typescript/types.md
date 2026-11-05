# TypeScript 类型

## 类型系统用法

### 不使用 `any`，未知类型用 `unknown` 【必须】

`any` 关闭检查并沿调用链扩散。类型未知时用 `unknown`，收窄后再使用。`{}` 同样不用，它表示任意非空值。

**例外：** 测试中构造替身时可以局部使用。

**工具：** `@typescript-eslint/no-explicit-any`、`no-unsafe-assignment`、`no-unsafe-member-access`。

**来源：** Google TypeScript 风格指南、typescript-eslint 文档

### 用 `@ts-expect-error`，不用 `@ts-ignore` 【必须】

问题修复后 `@ts-expect-error` 会报错提醒删除，`@ts-ignore` 会一直留着。原因写在指令后。

**工具：** `@typescript-eslint/ban-ts-comment`，配置见 [style.md](style.md)。

**来源：** Google TypeScript 风格指南、typescript-eslint 文档

### 用运行时判断收窄，不用断言 【推荐】

优先 `typeof`、`instanceof`、`in`、判别字段和相等比较。`as` 和 `!` 不产生运行时检查，错误会留到运行时。不用 `as unknown as T` 绕过检查。

**示例：**

反例：

```ts
function readName(value: unknown): string {
  return (value as User).name!;
}
```

正例：

```ts
function readName(value: unknown): string {
  if (typeof value === 'object' && value !== null && 'name' in value && typeof value.name === 'string') {
    return value.name;
  }
  throw new TypeError('value 缺少 name');
}
```

**例外：** `as const` 不是不安全断言。局部事实确实保证类型时可以用 `as` 或 `!`，理由就近写一句注释。

**工具：** `@typescript-eslint/no-non-null-assertion`、`no-unnecessary-type-assertion`。

**来源：** Google TypeScript 风格指南、TypeScript 手册

### 类型谓词必须真正检查 【推荐】

`x is T` 和 `asserts x is T` 是作者对编译器的承诺，编译器不验证函数体。谓词要覆盖 `T` 用到的全部字段，复杂结构改用校验库。

**来源：** TypeScript 手册

### 对象字面量用类型标注，不用 `as` 【推荐】

`const foo: Foo = {...}` 会报多余字段和字段缺失，`{...} as Foo` 不会。既要检查又要保留字面量类型时用 `satisfies`。

**工具：** `@typescript-eslint/consistent-type-assertions`。

**来源：** Google TypeScript 风格指南

### 用可辨识联合表达互斥状态 【推荐】

让非法组合无法被表示，不用多个可选字段拼状态。

**示例：**

```ts
type LoadState =
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: User };
```

**来源：** TypeScript 手册

### 联合类型的分支要穷尽 【推荐】

`switch` 用 `never` 兜底。新增成员时编译器会指出所有漏处理的位置。

**示例：**

```ts
function describeState(state: LoadState): string {
  switch (state.status) {
    case 'loading':
      return '加载中';
    case 'error':
      return state.error.message;
    case 'success':
      return state.data.name;
    default: {
      const unhandled: never = state;
      throw new Error(`未处理的状态 ${JSON.stringify(unhandled)}`);
    }
  }
}
```

**工具：** `@typescript-eslint/switch-exhaustiveness-check`。

**来源：** TypeScript 手册、typescript-eslint 文档

### 用最简单的类型构造 【推荐】

对象结构用 `interface`，联合、元组和工具类型用 `type`。导出函数写明返回类型，局部变量交给推断。派生字段用 `Pick`、`Omit`、`Partial`，不手工复制。能用接口继承表达时，不写自定义映射类型和条件类型，它们难读，重命名工具也跟踪不到。

**工具：** `@typescript-eslint/consistent-type-definitions`。

**来源：** Google TypeScript 风格指南

## 空值

### 项目内统一空值语义 【推荐】

内部代码用 `undefined` 或省略属性表示“没有”，与 `Map.get`、可选参数和可选链一致。`null` 只用于外部协议明确需要的位置，比如 JSON 字段和数据库列，并在边界转换。

**原因：** 两种空值混用时，每个调用方都要同时判断两者。`JSON.stringify` 会丢弃值为 `undefined` 的属性、保留 `null`，二者在传输时含义不同。

### 类型别名不内含 `null` 或 `undefined` 【推荐】

在使用处写 `Order | undefined`，不写 `type MaybeOrder = Order | undefined`。空值就近处理，不穿过多层。

**来源：** Google TypeScript 风格指南

### 区分可选属性和 `undefined` 值 【推荐】

`exactOptionalPropertyTypes` 下，`name?: string` 不接受显式 `undefined`。确实需要传 `undefined` 时写 `name: string | undefined`，语义是“必须提供，可以为空”。

**来源：** TSConfig 参考

### 空值判断不靠真值 【必须】

`0`、`''`、`false` 是合法值时，`if (count)` 会把 `0` 当成缺失，`if (name)` 会把空字符串当成缺失。显式比较，或用 `?.` 和 `??`。判断 `null` 和 `undefined` 两者时可以写 `value == null`，这是 `==` 的唯一例外。

**工具：** `@typescript-eslint/strict-boolean-expressions`、`prefer-nullish-coalescing`，ESLint `eqeqeq` 配 `{ null: 'ignore' }`。

**来源：** TypeScript 手册、Google TypeScript 风格指南

## 泛型

### 类型参数至少关联两处 【推荐】

类型参数的作用是关联类型。只出现一次时改用具体类型或 `unknown`。只出现在返回值上的泛型等于隐藏的类型断言。

**示例：**

反例：

```ts
function parseJson<T>(text: string): T {
  return JSON.parse(text);
}
```

正例：

```ts
function parseJson<T>(text: string, schema: z.ZodType<T>): T {
  return schema.parse(JSON.parse(text));
}
```

**工具：** `@typescript-eslint/no-unnecessary-type-parameters`。

**来源：** typescript-eslint 文档、Google TypeScript 风格指南

### 用约束表达需要的能力 【推荐】

函数体要用到的成员写进 `extends`，键和对象的关系用 `keyof` 约束。不要先写无约束的 `T` 再在函数体内断言。

**示例：**

```ts
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}
```

**来源：** TypeScript 手册

## 枚举

### 优先字面量联合或 `as const` 对象 【推荐】

它们没有额外运行时语义。`enum` 不能被 Node.js 类型剥离处理，`erasableSyntaxOnly` 下也会报错。不使用 `const enum`，它与单文件转译冲突，对 JavaScript 使用方不可见。

**示例：**

```ts
const OrderStatus = { Pending: 'pending', Paid: 'paid' } as const;
type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
```

**例外：** 项目已统一使用 `enum` 时沿用，成员写字面量值。

**工具：** `@typescript-eslint/prefer-literal-enum-member`。

**来源：** TypeScript 手册、TSConfig 参考、Google TypeScript 风格指南

## 数据类

### 纯数据用接口，不用类 【推荐】

类型系统是结构化的，普通对象满足字段就能赋给类类型，类不提供额外保证。只有带行为和不变量时才写类。不会修改的字段用 `readonly`，入参用 `readonly T[]`。

**来源：** Google TypeScript 风格指南

### 同形不同义的标识用品牌类型 【推荐】

结构化类型下 `userId` 和 `orderId` 都是 `string`，可以互相传错。混用会造成缺陷时，用 `string & { readonly __brand: 'UserId' }`，并只在校验函数中创建。

## 类型边界校验

### 外部数据在边界校验 【必须】

类型在运行时不存在。请求体、接口响应、存储内容和 URL 参数都按 `unknown` 接收，用 Zod、Valibot 等校验后再进入业务层，内部层不再重复判断。类型从 schema 推导，只保留一份定义。`process.env` 在启动时一次解析成配置对象，失败立即退出。

**示例：**

```ts
const orderSchema = z.object({ id: z.string(), totalCents: z.number().int() });
type Order = z.infer<typeof orderSchema>;

const order: Order = orderSchema.parse(await response.json());
```

**工具：** `no-unsafe-assignment` 会报告 `response.json()` 返回的 `any`。

**来源：** MDN
