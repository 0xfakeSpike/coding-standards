# TypeScript / JavaScript 风格

适用于 Node.js 服务、库和脚本。React、Vue、CSS 见前端层。

## 工具配置

### tsconfig 基线 【推荐】

新项目默认使用下面的配置。`strict` 不包含这里的其他检查项，要逐项开启。`module`、`rootDir`、`types` 按项目环境调整。`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes` 代价较高，可以按项目关闭。开启时遍历优先用 `for...of` 和 `entries()`，不写下标访问。

**原因：** `noUncheckedIndexedAccess` 让越界访问在编译期暴露，`exactOptionalPropertyTypes` 区分“没有这个属性”和“值为 `undefined`”，`verbatimModuleSyntax` 让导入语句按写法输出，单文件转译工具也能得到一致结果。

**示例：**

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "module": "nodenext",
    "target": "es2023",
    "rootDir": "./src",
    "outDir": "./dist",
    "types": ["node"]
  },
  "include": ["src"]
}
```

**例外：** 使用打包器的项目把 `module` 设为 `preserve` 或 `esnext`，`moduleResolution` 设为 `bundler`。

**工具：** `tsc --noEmit`，纳入 CI。

**来源：** TSConfig 参考、TypeScript 手册

### 用 typed lint 预设 【推荐】

**适用：** 多人协作

ESLint 使用 typescript-eslint 的 `recommendedTypeChecked`，团队能接受时升级到 `strictTypeChecked`。`no-floating-promises`、`no-misused-promises`、`await-thenable` 等规则依赖类型信息，无类型预设不会启用。

**工具：** `eslint` 或 Biome，纳入 CI。

**来源：** typescript-eslint 文档

### lint 配置不要求文档注释 【必须】

不启用 `jsdoc/require-jsdoc` 这类要求文档注释的规则，对外发布的库除外。`@ts-expect-error` 后写一句原因，`ban-ts-comment` 默认就这样要求。

**来源：** 团队

### 业务代码不留 `console.log` 【推荐】

需要保留的输出走项目的日志模块，并开启 ESLint `no-console`。命令行工具的正常输出除外。

**原因：** 调试输出绕过日志级别和脱敏，容易把敏感数据留在线上。

## 格式

### 格式交给格式化工具 【推荐】

**适用：** 多人协作

使用 Prettier 或 Biome，全项目一种。不启用 ESLint 的格式类规则，评审不讨论格式。不依赖自动分号插入，字符串用单引号。配置写进仓库并在 CI 中检查。

**工具：** `prettier --check .` 或 `biome check`。

**来源：** typescript-eslint 文档、Google TypeScript 风格指南

### 注释只写代码表达不了的原因 【必须】

不写复述代码的注释，包括只重复名字和类型的文档注释，对外发布的库的 API 文档和许可证头除外。需要靠注释才能读懂时，改进命名和结构。工具指令不算注释，比如 `@ts-expect-error`、`eslint-disable-next-line`。代码表达不了的原因就近写一句，比如外部系统的限制、反直觉的行为，以及忽略错误、类型断言、抑制 lint 这类单点取舍。

**来源：** 团队

## 命名

### 按标识符类别使用大小写 【推荐】

| 对象 | 规则 | 示例 |
|---|---|---|
| 变量、函数、方法、属性 | `camelCase` | `orderTotal`、`fetchUser` |
| 类、接口、类型、枚举 | `PascalCase` | `OrderService` |
| 模块级常量 | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| 泛型参数 | `T` 或 `PascalCase` 描述名 | `TItem` |
| 文件 | `kebab-case`，或沿用项目约定 | `order-service.ts` |

只有模块级、程序生命周期内不变的值才用 `UPPER_SNAKE_CASE`。接口不加 `I` 前缀，私有成员不加下划线。下划线前缀只用于语法上必须存在但不使用的绑定。`number` 无法表达单位，写成 `timeoutMs`、`priceCents`。布尔值用 `is`、`has`、`can`、`should` 开头，函数以动词开头，集合用复数。

**工具：** `@typescript-eslint/naming-convention`。

**来源：** Google TypeScript 风格指南

### 缩写按单词处理 【推荐】

写 `loadHttpUrl`、`customerId`，不写 `loadHTTPURL`、`customerID`。平台名称除外，比如 `XMLHttpRequest`。

**来源：** Google TypeScript 风格指南

### 映射类变量写明 key 【必须】

`Map`、`Record` 和索引签名对象的名字要说明通过什么查找什么。索引签名的键也写有意义的名字。

**示例：**

反例：

```ts
const prices = new Map<string, number>();
const users: Record<string, User> = {};
```

正例：

```ts
const priceBySku = new Map<string, number>();
const userById = new Map<string, User>();
```

**来源：** 团队

## 导入排序

### 导入顺序交给工具 【推荐】

顺序统一为：Node 内置模块、第三方包、项目别名、相对路径。内置模块写 `node:` 前缀。排序由 Biome 或 ESLint 插件自动完成，不手工调整。

### 仅类型的导入写 `import type` 【推荐】

`verbatimModuleSyntax` 下不写 `type` 的导入会保留到运行时。Node.js 类型剥离时，这种写法会变成运行时错误。

**工具：** `@typescript-eslint/consistent-type-imports`。

**来源：** TSConfig 参考、Node.js API 文档

### 优先具名导出 【推荐】

默认导出没有规范名称，导入方可以任意命名，拼错时也不报错。框架要求默认导出时除外。

**工具：** `import-x/no-default-export`。

**来源：** Google TypeScript 风格指南

## 文件组织

### 只导出模块外部用到的符号 【推荐】

不用 `export let`，可变状态通过函数访问。不写只含静态成员的容器类，文件本身就是命名空间。不用 `namespace`。

**来源：** Google TypeScript 风格指南

### 不引入循环依赖 【必须】

循环依赖会造成初始化顺序问题，读到未初始化的绑定。`export *` 全量转出是常见来源，还会拖慢构建，不使用。显式列出导出的模块入口可以使用，比如 `export { createOrder } from './create-order.js'`。

**工具：** `import-x/no-cycle` 或 `madge --circular`。

**来源：** NestJS 文档
