# 构建与浏览器兼容

本文以 Vite 为例，讲构建配置、环境变量和浏览器支持。其他构建工具遵循同样的要求。构建可复现和发布流程见 [发布](../general/release.md)，加载性能见 [性能](performance.md)，Source Map 见 [安全](security.md)，TypeScript 配置见 [TypeScript 项目配置](../languages/typescript/project.md)。

## 构建配置

### Vue 项目用 `vue-tsc` 做类型检查 【推荐】

Vite 只转译 `.ts` 文件，不做类型检查。类型检查和单文件转译的配置见 [project.md](../languages/typescript/project.md)，Vue 项目用 `vue-tsc --noEmit`。

**来源：** Vite 文档

### 动态路径使用 `import.meta.env.BASE_URL` 【推荐】

部署到子路径时配置 `base`。运行时拼接资源地址用 `import.meta.env.BASE_URL`。这个变量在构建时静态替换，`import.meta.env['BASE_URL']` 这种写法不会被替换。

**来源：** Vite 文档

Source Map 的配置见 [security.md](security.md)。

## 环境变量

### 前端环境变量视为公开信息 【必须】

`VITE_*`、`NEXT_PUBLIC_*` 都会被打包进产物，绝不放密钥。不把 `envPrefix` 设成空字符串。`.env.*.local` 只存本地值，`.gitignore` 要包含 `*.local`。

**来源：** Vite 文档

### 声明变量类型并转换 【推荐】

在 `vite-env.d.ts` 中扩展 `ImportMetaEnv`。环境变量的值都是字符串，数字和布尔值使用前要转换。

反例：

```ts
if (import.meta.env.VITE_ENABLE_CHAT) {
  startChat();
}
```

正例：

```ts
if (import.meta.env.VITE_ENABLE_CHAT === 'true') {
  startChat();
}
```

**原因：** 字符串 `'false'` 也是真值。

**来源：** Vite 文档

### 公开变量在构建时固化 【推荐】

环境变量在构建时写入产物，修改后要重新构建。同一产物部署到多个环境时，由服务端下发配置。

**来源：** Vite 文档、Next.js 文档

## 代码分割

### 依靠动态导入分割 【推荐】

路由的分割见 [路由](routing.md)，大型依赖见 [性能](performance.md)。框架提供分割配置时用框架的方式。需要手动拆包时用构建工具的官方选项，比如 Vite 的 `build.rolldownOptions.output.codeSplitting`，需要 Vite 8 及以上。

**来源：** Vite 文档

### 不调高告警阈值掩盖大文件 【推荐】

`build.chunkSizeWarningLimit` 告警时，先分析并拆分产物。这个限制按未压缩体积比较，因为脚本体积与执行时间相关。

**来源：** Vite 文档

## 浏览器支持

### 明确声明支持的浏览器 【推荐】

把支持范围写成 Browserslist 配置，放在 `.browserslistrc` 或 `package.json`。范围不清楚时从 `defaults` 开始。

```text
defaults and fully supports es6-module
```

**原因：** 没有声明时，各工具使用各自的默认值，产物和预期不一致。

**来源：** Browserslist 文档

### 不用单一条件定义范围 【推荐】

需要调整时组合使用，比如 `last 2 versions, not dead, > 0.2%`。`last n versions` 单独使用会漏掉仍然流行的旧版本。直接指定浏览器只用于固定环境。用 `update-browserslist-db` 更新数据，用 `npx browserslist` 确认实际范围。

**来源：** Browserslist 文档

### 构建目标与声明一致 【推荐】

Vite 的 `build.target` 不读取 Browserslist，默认目标是发布时固定的 Baseline Widely Available。声明的范围低于默认目标时，要显式设置 `build.target`。

```ts
export default defineConfig({
  build: { target: ['chrome100', 'safari15', 'firefox100'] },
});
```

**来源：** Vite 文档

### 用特性检测，不判断 User-Agent 【推荐】

JavaScript 检查能力是否存在，CSS 用 `@supports`。

反例：

```ts
if (navigator.userAgent.includes('Safari')) {
  useFallbackShare();
}
```

正例：

```ts
if ('share' in navigator) {
  await navigator.share(data);
}
```

**来源：** MDN

## Polyfill

### 语法转换不等于 Polyfill 【必须】

Vite 默认只转换语法，不注入 Polyfill。使用了目标浏览器缺少的 API 时，要显式提供 Polyfill。

**来源：** Vite 文档

### 按目标浏览器按需引入 【推荐】

Babel 项目用 core-js 按需注入，配置写法以所用 Babel 版本的文档为准。`corejs` 写明次版本号，只写主版本时次版本新增的模块不会注入。这种模式下不手动导入 core-js。发布给别人使用的库不污染全局，用 `core-js-pure`。

**来源：** core-js 文档

### 低版本浏览器用官方插件 【推荐】

需要支持没有原生 ESM 的浏览器时，用 `@vitejs/plugin-legacy` 生成单独的产物，只在这类浏览器中加载。未设置 `targets` 时它读取 Browserslist 配置。未设置 `modernTargets` 时不用 `modernPolyfills: true`，自动检测注入的内容过多。插件依赖内联脚本，使用严格 CSP 时用导出的 `cspHashes` 生成响应头。

**来源：** Vite 文档

## 发布缓存

### 部署时先传资源再切 HTML 【推荐】

**适用：** 线上服务

产物文件名保留内容 hash，响应头见 [性能](performance.md)。部署时先上传静态资源，再切换 HTML。

**原因：** 先切换 HTML 时，新页面引用的资源可能还不存在。

**来源：** web.dev HTTP 缓存

发布后旧分块加载失败的处理见 [路由](routing.md)。
