# 前端状态管理

本文规定状态放在哪里、Store 怎么设计、哪些状态要持久化或进入 URL。请求缓存见 [data-fetching.md](data-fetching.md)，表单状态见 [forms.md](forms.md)，组件内部的 UI 状态见 [components.md](components.md)。

## 状态分层

### 按状态性质选择存放位置 【推荐】

不把所有数据都放进全局 Store。局部状态留在最近的组件，多个不相邻的组件确实要共享时才提升。

| 状态类型 | 示例 | 存放位置 |
|---|---|---|
| 服务端数据 | 列表、详情 | TanStack Query、SWR、Pinia Colada、Next.js Server Component、Nuxt `useFetch` 等 |
| 页面可分享状态 | 筛选、分页、Tab | URL 查询参数 |
| 局部 UI 状态 | 弹窗开关、输入值 | 组件内部 |
| 跨组件客户端状态 | 主题、当前用户 | Zustand、Pinia、Context |
| 表单状态 | 字段、校验 | React Hook Form、VeeValidate 等 |

**来源：** Redux 风格指南

### 同一份数据只保留一个权威来源 【必须】

服务端数据留在请求缓存，URL 状态留在 URL，其他位置只读取，不复制。登录用户这类全局会话数据可以由 Store 持有，它就是这份数据的权威来源，请求缓存不再保存一份。

**原因：** 复制之后就要手工维护同步，两份数据迟早不一致。

**示例：**

反例：

```ts
useEffect(() => {
  if (data) useOrderStore.getState().setOrder(data);
}, [data]);
```

正例：

```ts
const { data: order } = useQuery(orderQuery(id));
```

### 服务端数据使用数据请求库 【推荐】

缓存、去重、重试、失效更新自己写既费时又容易出错，交给请求库，不写在 Store 里。

**来源：** Redux 风格指南、TanStack Query 文档

## Store 设计

### Store 按业务域拆分 【推荐】

按数据类型或业务域划分，比如 `auth`、`cart`，不按页面划分，也不做包含所有内容的巨型 Store。Pinia 每个业务域一个 Store；Zustand 和 Redux 官方推荐单一 Store，内部用 slice 按业务域拆分。

**原因：** 同一份数据常被多个页面读取，按页面划分会造成重复和归属不清。

**来源：** Redux 风格指南、Zustand 文档、Pinia 文档

### 状态更新集中在 Store 的动作里 【推荐】

动作和状态放在一起，组件只调用动作。Zustand 用 `set` 更新，Redux 不直接修改状态，action 按发生的事件命名。

**原因：** 更新分散到组件后，无法定位状态被谁改变，订阅者也可能收不到通知。

**来源：** Redux 风格指南、Zustand 文档

### 状态字段使用可序列化的普通数据 【推荐】

状态字段不放 Promise、类实例和 DOM 节点，否则调试工具和持久化会失效。Redux 的 state 和 action 必须遵守，也不放 `Map`、`Set`。Zustand 和 Pinia 中的动作函数不属于状态字段。

**来源：** Redux 风格指南

### 订阅时只选择需要的字段 【推荐】

组件不订阅整个 Store。Zustand 的选择器返回新数组或新对象时，用 `useShallow` 包裹。Pinia 的解构见 [vue.md](vue.md)。

**示例：**

```ts
const { items, total } = useCartStore(
  useShallow((state) => ({ items: state.items, total: state.total })),
);
```

**来源：** Zustand 文档、Pinia 文档、Redux 风格指南

Pinia 的约束见 [vue.md](vue.md)。

## 选型

### 每一层在项目内只用一种方案 【推荐】

| 场景 | 选择 |
|---|---|
| Vue 客户端状态 | Pinia |
| React 中小型客户端状态 | Zustand |
| React 已使用 Redux | Redux Toolkit，服务端数据用 RTK Query |
| 低频变化的依赖注入，比如主题 | Context、`provide` |

**来源：** Redux 风格指南、Pinia 文档

## 派生状态

### 能推导的值不存储 【必须】

只保存原始数据，总价、过滤结果、是否全选都在读取时计算。高开销的计算要记忆化：Redux 用 `createSelector`，选择器命名为 `selectXxx`；Vue 用 `computed` 或 Pinia getter；React 组件内的写法见 [react.md](react.md)。

**原因：** 存下来的派生值要在每个写入点同步，漏一处就出错。

**来源：** Redux 风格指南

## 持久化

### 只持久化刷新后仍需要的字段 【推荐】

Zustand 的 `persist` 用 `partialize` 选择。敏感信息不进浏览器存储，见 [security.md](security.md)。

**来源：** Zustand 文档

### 读写存储集中在一处并容错 【推荐】

`localStorage` 的访问和写入集中在一个模块，放在 `try/catch` 中，读取后校验格式，失败时按无数据处理。

**原因：** 存储被禁用时访问会抛 `SecurityError`，空间已满时写入会抛 `QuotaExceededError`，已有数据也可能格式不符。

**示例：**

```ts
function readTheme(): Theme | null {
  try {
    const raw = localStorage.getItem('myapp:theme');
    const parsed = themeSchema.safeParse(raw === null ? null : JSON.parse(raw));
    return parsed.success ? parsed.data : null;
  } catch {
    return null;
  }
}
```

**来源：** MDN、Zod 文档

### 存储 key 加项目前缀 【推荐】

比如 `myapp:theme`，避免同域名下的应用互相覆盖。

**来源：** Zustand 文档

### 处理存储的恢复时机和跨页同步 【推荐】

服务端读不到浏览器存储，首屏用初始值渲染，挂载后再恢复，Zustand 用 `skipHydration` 配合 `rehydrate()`。跨标签页同步监听 `storage` 事件，该事件不在发起修改的页面触发。

**来源：** Zustand 文档、MDN

## URL 状态

### 可分享的页面状态放在 URL 【推荐】

筛选、页码、排序、Tab 放进查询参数，Store 不再保存一份。接口用游标分页时，游标不放进 URL，过滤条件变化时清除游标。

**原因：** 放在内存里时，刷新、分享链接和后退都会丢失状态。

### URL 参数按外部数据校验 【必须】

URL 可以被用户任意修改。读取时解析并校验，非法值回退到默认值，规则见 [types.md](../languages/typescript/types.md)。

**示例：**

```ts
const listParamsSchema = z.object({
  page: z.coerce.number().int().positive().catch(1),
  status: z.enum(['open', 'closed']).catch('open'),
});
```

**来源：** Zod 文档

### 按后退预期选择 push 或 replace 【推荐】

翻页和 Tab 这类用户希望能后退的切换用 push。输入中持续变化的参数用 replace，比如 `setSearchParams({ keyword }, { replace: true })`。React Router 的 `setSearchParams` 不排队，多个参数要在一次调用中更新。

**来源：** MDN、React Router 文档

### URL 不放敏感信息 【必须】

密码、令牌、个人信息不进入路径和查询参数，它们会留在历史记录和日志中，也会随分享扩散。

**来源：** web.dev 学习表单
