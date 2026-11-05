# TypeScript / JavaScript 惯用写法

## 惯用写法

### 不写只转调一次的函数和无意义的封装 【必须】

只把调用原样转发一次的函数、类和模块直接删除，调用方使用原始 API。

**示例：**

反例：

```ts
function getUser(id: string): Promise<User> {
  return userRepository.findById(id);
}
```

正例：

```ts
const user = await userRepository.findById(id);
```

**来源：** 团队

### 默认 `const`，不用 `var` 【必须】

需要重新赋值时才用 `let`。每条声明只声明一个变量。

**工具：** ESLint `prefer-const`、`no-var`。

**来源：** Google TypeScript 风格指南

### 使用 `===` 和 `!==` 【必须】

`==` 的隐式转换难以预测。例外见 [types.md](types.md) 的空值判断。

**工具：** ESLint `eqeqeq`。

**来源：** Google TypeScript 风格指南

### 默认值用 `??`，不用 `||` 【必须】

`||` 会把 `0`、`''`、`false` 这些合法值替换掉。写 `options.pageSize ?? 20`。

**工具：** `@typescript-eslint/prefer-nullish-coalescing`。

### 命名函数用函数声明 【推荐】

顶层命名函数写 `function`，回调用箭头函数。需要类型标注时才用 `const fn: Handler = () => {}`。不用 `this` 重绑定，用箭头函数或显式参数。

**来源：** Google TypeScript 风格指南

### 调用处读不懂时用对象参数 【推荐】

`createUser(data, true)` 看不出 `true` 的含义。写成 `createUser(data, { sendWelcomeMail: true })`，或拆成两个函数。

### 不修改入参 【推荐】

用展开、`map`、`filter` 和 `toSorted` 产生新值。展开是浅拷贝，深拷贝用 `structuredClone`，它不能复制函数。

**来源：** MDN

## 集合

### 动态键用 `Map`，集合用 `Set` 【推荐】

键来自运行时数据时用 `Map`。普通对象会暴露原型链上的属性，键也只能是字符串或 Symbol。`Record` 用于键集合静态已知的情况。

**来源：** Google TypeScript 风格指南、MDN

## 字符串

### 组合字符串用模板字符串 【推荐】

不用 `+` 拼接多段内容。模板中只放 `string` 和 `number`，对象要先格式化。

**工具：** `@typescript-eslint/restrict-template-expressions`、`restrict-plus-operands`。

**来源：** typescript-eslint 文档

### `length` 不等于字符数 【必须】

`length` 是 UTF-16 码元数。截断或统计用户文本时用 `Intl.Segmenter` 按字素处理，否则 emoji 和组合字符会被切断。面向用户的排序用 `Intl.Collator`，`<` 只按码元比较。

## 日期时间

### 时间点用 UTC 存储和传输 【必须】

存储和接口使用 ISO 8601 字符串或时间戳，时区转换放在展示层。`Date` 内部是 UTC 时间戳，但 `getHours` 等方法按进程所在时区解释。

**来源：** MDN

### 明确 `Date` 的解析规则 【必须】

`new Date('2026-01-01')` 按 UTC 解析，`new Date('2026-01-01T00:00')` 按本地时间解析。月份从 0 开始。setter 会修改原对象。解析外部格式时用校验过的 ISO 8601 字符串，不依赖非标准格式。

**来源：** MDN

### 优先 `Temporal` 【推荐】

运行时支持时用 `Temporal` 处理时区、日历和日期运算。不支持时用成熟的日期库，不手写时区和月份运算。类型存在不代表运行时可用。

**来源：** MDN、TypeScript 发布说明

## 数值与金额

### 金额不用 `number` 做小数运算 【必须】

`number` 是 IEEE 754 双精度浮点数，`0.1 + 0.2 !== 0.3`。用最小单位的整数或 decimal 库，名字带单位，比如 `totalCents`。

**来源：** MDN

### 大整数不用 `number` 【必须】

超过 `Number.MAX_SAFE_INTEGER`（2^53 - 1）后精度丢失。数据库的 64 位 ID 用字符串传输，运算用 `bigint`。`JSON.stringify` 遇到 `bigint` 会抛错，`JSON.parse` 读大数字时会丢精度。

**来源：** MDN

### 字符串转数字后检查结果 【必须】

用 `Number(text)` 并检查 `Number.isFinite`。`parseInt('12px')` 返回 12，只在需要非十进制进制时使用 `parseInt`，并写明进制。空字符串经 `Number` 转换得到 0，需要单独判断。

**来源：** Google TypeScript 风格指南、MDN

