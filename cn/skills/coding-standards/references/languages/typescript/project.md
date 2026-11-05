# TypeScript / JavaScript 项目结构

## 目录结构

### 按功能组织源码 【推荐】

`src/` 放源码，`dist/` 放产物且不提交。一个功能的路由、服务和类型放在一起，不按 `controllers/`、`services/` 这样的技术类型铺满全局。Express、Fastify、NestJS 的目录写法见 [node.md](node.md) 的“按框架组织功能目录”。

**示例：**

```text
src/
  orders/
    order-service.ts
    order-service.test.ts
    order-routes.ts
  config.ts
  server.ts
```

### 应用实例和监听分开 【推荐】

`app.ts` 只构建应用，`server.ts` 负责 `listen`，这样测试可以不占端口。

**来源：** Fastify 文档

## 包管理

### 全项目只用一种包管理器 【推荐】

提交对应的 lock 文件，不混用多种 lock 文件。

### CI 用锁定安装 【推荐】

**适用：** 多人协作

CI 和镜像构建用 `npm ci` 或 `pnpm install --frozen-lockfile`。lock 文件和 `package.json` 不一致时直接失败，不更新 lock 文件。

**来源：** npm 文档、pnpm 文档

### 依赖分类准确 【推荐】

运行时需要的放 `dependencies`，类型包、构建和测试工具放 `devDependencies`。代码用到的包必须直接声明，不依赖传递依赖。

### 限制依赖的安装脚本 【必须】

**适用：** 线上服务

pnpm 用 `allowBuilds` 列出允许执行构建脚本的依赖，npm 用 `--ignore-scripts` 或 `ignore-scripts` 配置。

**来源：** Node.js 安全最佳实践、pnpm 文档

## 版本与锁定

### 声明运行时版本 【推荐】

`package.json` 写 `engines.node`，本地、CI 和镜像使用同一个主版本，选择 LTS 版本。`engines` 默认只产生警告，要强制时配置 `engine-strict`。

**来源：** npm 文档

### TypeScript 版本满足工具链要求 【推荐】

升级 TypeScript 前核对 lint 等工具的版本要求。工具暂不支持时临时固定在兼容版本，记录解除条件，工具支持后升级。不同时安装两个版本。

**来源：** typescript-eslint 文档

### 不提交未使用的类型包 【推荐】

`types` 明确列出全局类型后，`@types/*` 只保留实际需要的。

## 构建

### 新项目默认 ESM 【推荐】

Node.js 项目在 `package.json` 写 `"type": "module"`，`module` 用 `nodenext`。

| 项 | ESM | CommonJS |
|---|---|---|
| 判定 | `.mjs`，或 `"type": "module"` 下的 `.js` | `.cjs`，或 `"type": "commonjs"` 下的 `.js` |
| 相对导入 | 必须写扩展名 | 可省略 |
| 当前文件路径 | `import.meta.dirname`、`import.meta.filename` | `__dirname`、`__filename` |
| 顶层 `await` | 支持 | 不支持 |

**例外：** 打包器项目是否设置 `"type": "module"` 按打包器文档决定。

**来源：** Node.js API 文档、TypeScript 手册

### `nodenext` 下相对导入写输出文件的扩展名 【推荐】

源码是 `./order.ts`，导入写 `./order.js`。TypeScript 不改写普通导入路径，省略扩展名在运行时报错。直接运行 `.ts` 时写 `./order.ts` 并开启 `allowImportingTsExtensions`，同时要产出 JavaScript 时开启 `rewriteRelativeImportExtensions`。

**来源：** TypeScript 手册、TSConfig 参考、Node.js API 文档

### 明确 `require(esm)` 的限制 【推荐】

`require()` 加载 ESM 需要运行时支持。被加载的模块及其依赖不能使用顶层 `await`。

**来源：** Node.js API 文档

### 路径别名用 `imports` 【推荐】

`package.json` 的 `imports` 由 Node.js 和 TypeScript 共同理解。`tsconfig` 的 `paths` 不改写输出，运行时找不到，类型剥离下直接报错。`#/` 前缀需要 TypeScript 6.0 和 Node.js 24.14+ 或 25.4+。

**示例：**

```json
{
  "type": "module",
  "imports": {
    "#/*": "./dist/*"
  }
}
```

**来源：** Node.js API 文档、TypeScript 发布说明

### 类型检查是独立步骤 【推荐】

**适用：** 多人协作

esbuild、tsx 和 Node.js 类型剥离都不检查类型。CI 单独运行 `tsc --noEmit`，范围包含测试文件。使用这类工具时开启 `verbatimModuleSyntax` 或 `isolatedModules`，让编译器报告单文件转译处理不了的写法。

**来源：** TSConfig 参考、Node.js API 文档

### 直接运行 `.ts` 时只用可擦除语法 【推荐】

类型剥离需要 Node.js 22.18 及以上。它不读 `tsconfig.json`，不支持 `enum`、带运行时代码的 `namespace` 和参数属性。适合脚本和工具。开启 `erasableSyntaxOnly` 让编译器提前报错。

**例外：** NestJS 依赖 `experimentalDecorators` 和 `emitDecoratorMetadata`，必须走编译，不开 `erasableSyntaxOnly`。

**来源：** Node.js API 文档、TSConfig 参考、NestJS 文档

### 依赖包不发布 TypeScript 源码 【必须】

Node.js 拒绝处理 `node_modules` 下的 `.ts` 文件。发布编译后的 JavaScript 和 `.d.ts`。

**来源：** Node.js API 文档

### 用 `exports` 声明包的公共入口 【推荐】

库用 `exports` 列出入口，未列出的路径不能被导入。发布范围用 `files` 白名单控制，发布前用 `npm publish --dry-run` 检查。私有包写 `"private": true`。

**来源：** Node.js API 文档、Node.js 安全最佳实践、npm 文档

## 多模块或 monorepo

### 用 workspace 管理多包 【推荐】

使用 pnpm workspace，内部依赖写 `workspace:*`，保证解析到本地包。共用依赖的版本用 `catalog:` 统一。

**来源：** pnpm 文档

### 包之间只通过入口依赖 【推荐】

不用相对路径跨包导入，不引用其他包的 `src` 内部文件。

### 用项目引用组织类型检查 【推荐】

被引用的包开启 `composite`，根配置列出 `references`，用 `tsc -b` 增量构建。这样能强制包之间的依赖方向，也能缩短检查时间。

**来源：** TypeScript 手册

### 公共配置只保留一份 【推荐】

根目录放 `tsconfig.base.json`，各包用 `extends` 继承，只写差异。ESLint 和格式化配置也放根目录。
