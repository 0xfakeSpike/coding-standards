# Nuxt

先阅读 [vue.md](vue.md)。服务端接口的通用规范见 [后端层](../backend/index.md)。取数以本文为准，优先于 [data-fetching.md](data-fetching.md)。

## 目录约定

### 按约定目录放代码 【推荐】

下表的 `app/` 目录结构需要 Nuxt 4 及以上。

| 目录 | 内容 |
|---|---|
| `app/pages/` | 文件路由 |
| `app/components/`、`app/composables/`、`app/utils/` | 组件和客户端逻辑，自动导入 |
| `app/layouts/` | 页面布局 |
| `app/middleware/` | 路由导航前运行的代码 |
| `app/plugins/` | Nuxt 应用创建时运行的代码 |
| `server/` | 服务端路由和逻辑 |
| `shared/` | 应用和服务端都能用的代码 |
| `public/` | 不经构建的静态文件 |

按业务功能组织时，在这些目录下按功能建子目录，或拆成 Nuxt Layers。

**来源：** Nuxt 文档

### 应用代码和服务端代码不互相导入 【必须】

两边都要用的类型和纯函数放进 `shared/`。

**原因：** 两边运行环境不同，混用会把服务端代码带进客户端。

**来源：** Nuxt 文档

### Composable 只在 Nuxt 上下文中同步调用 【必须】

只在 `<script setup>`、插件和路由中间件里调用，不放在文件顶层。普通函数中不能在 `await` 之后调用。

**原因：** 否则会报 Nuxt 实例不可用。

**来源：** Nuxt 文档

### 用 `routeRules` 按路由设置渲染方式 【推荐】

选型依据见 [architecture.md](architecture.md)。

```ts
export default defineNuxtConfig({
  routeRules: {
    '/pricing': { prerender: true },
    '/blog': { swr: 3600 },
    '/dashboard': { ssr: false },
  },
});
```

**来源：** Nuxt 文档

## 数据获取

### 首屏数据用 `useFetch` 或 `useAsyncData` 【必须】

普通接口用 `useFetch`，组合多个请求或使用第三方查询层时用 `useAsyncData`。写 `await useFetch('/api/orders')`，不在 `setup` 中直接 `await $fetch('/api/orders')`。用户操作触发的请求才用 `$fetch`。需要统一 baseURL、鉴权和错误转换时，用 `$fetch.create` 建实例，配合 `createUseFetch` 或 `useAsyncData` 使用，代替统一请求客户端。

**原因：** `$fetch` 在服务端和客户端各请求一次，还可能导致水合问题。这两个函数会把服务端结果带到客户端。

**来源：** Nuxt 文档

### `useAsyncData` 的 key 唯一，相同 key 的选项一致 【必须】

相同 key 的调用共享 `data`、`error` 和 `status`。`deep`、`transform`、`pick`、`default` 等选项必须一致，否则用不同的 key。

**来源：** Nuxt 文档

### `lazy` 和 `server: false` 要处理等待状态 【必须】

这两种方式在渲染时可能还没有数据，要根据 `status` 显示加载界面。

**来源：** Nuxt 文档

### 用 `pick` 或 `transform` 缩小负载 【推荐】

它们减少发送到客户端的数据，但不减少首次请求的数据量。

**来源：** Nuxt 文档

### 服务端发起的请求要带上用户身份 【必须】

`useFetch` 使用相对地址时会自动转发请求头和 Cookie。使用 `$fetch` 时，手工通过 `useRequestHeaders(['cookie'])` 传递，或改用 `useRequestFetch`。

**原因：** 否则接口在服务端渲染时拿不到登录状态。

**来源：** Nuxt 文档

## 服务端路由

### 用文件名表达路径和方法 【推荐】

`server/api/` 下的文件映射到 `/api`，其他路由放 `server/routes/`。文件名加方法后缀，比如 `orders.get.ts`。服务端路由之间调用用 `event.$fetch`，全局 `$fetch` 不转发入站请求头和上下文。

**来源：** Nuxt 文档

### 入参用带校验的读取函数 【必须】

使用 `getValidatedQuery`、`getValidatedRouterParams`、`readValidatedBody`，配合 Schema 校验。直接传 Schema 时校验失败返回 400，而 [rest.md](../backend/api/rest.md) 和前端的字段错误映射都按 422 处理，所以传 `safeParse` 后自己返回 422。

```ts
export default defineEventHandler(async (event) => {
  const result = await readValidatedBody(event, orderSchema.safeParse);
  if (!result.success) {
    throw createError({
      status: 422,
      statusText: 'Validation Failed',
      data: { errors: toFieldErrors(result.error) },
    });
  }
  return createOrder(result.data);
});
```

**来源：** Nuxt 文档

### 用 `createError` 返回错误状态 【必须】

未捕获的错误都会返回 500。要返回其他状态时写 `throw createError({ status: 404, statusText: 'Order Not Found' })`，状态码的含义按 [rest.md](../backend/api/rest.md)。

**来源：** Nuxt 文档

### 服务端中间件不返回响应 【必须】

`server/middleware/` 在每个请求前运行，只检查或扩展 `event.context`，或抛出错误。

**来源：** Nuxt 文档

## 运行时配置

### 配置项在 `runtimeConfig` 中声明 【必须】

顶层键只在服务端可用，`public` 下的键会发送到客户端。机密不放进 `public`，也不传给 `useState` 或模板。

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '',
    public: { apiBase: '' },
  },
});
```

**来源：** Nuxt 文档

### 环境变量按 `NUXT_` 规则命名 【必须】

大写、`NUXT_` 开头，用下划线分隔层级，比如 `NUXT_PUBLIC_API_BASE` 覆盖 `runtimeConfig.public.apiBase`。不符合规则的变量运行时不生效。构建后的服务不读 `.env` 文件，要由运行环境提供变量。

**来源：** Nuxt 文档

### 配置值必须可序列化 【必须】

函数、`Set`、`Map` 不放进配置，相关逻辑写进插件或中间件。纯数字或类 JSON 的机密要加引号，否则会被转成其他类型。服务端路由中用 `useRuntimeConfig(event)`，运行时环境变量覆盖才会生效。

**来源：** Nuxt 文档

## 状态与水合

### 跨组件的 SSR 状态不放模块级变量 【必须】

用 `useState` 或 Pinia，不在模块顶层导出 `ref`。比如 `export const useUser = () => useState<User | undefined>('user')`。`useState` 的值必须能序列化成 JSON。

**原因：** 模块级状态在服务端被所有请求共享，会泄漏数据。

**来源：** Nuxt 文档

### 用 SSR 友好的方式消除水合差异 【必须】

SSR 首屏就要用到的小型持久值用 `useCookie`，其余浏览器存储按 [state-management.md](state-management.md) 在挂载后恢复。只能在浏览器渲染的内容放进 `ClientOnly`，操作 DOM 的第三方库在 `onMounted` 中初始化。

**来源：** Nuxt 文档

### Pinia 通过 `@pinia/nuxt` 接入 【推荐】

会话数据放 Store 时，页面中写 `await callOnce('user', () => store.fetchUser())` 调用取数 Action，避免重复请求。其余首屏数据用 `useFetch`，Store 的边界见 [state-management.md](state-management.md)。

**来源：** Pinia 文档
