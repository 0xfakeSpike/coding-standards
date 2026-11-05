# 前端数据请求

本文规定浏览器端请求的客户端、缓存、竞态、重试、乐观更新、上传下载和错误分类。Next.js 的服务端取数见 [nextjs.md](nextjs.md)，Nuxt 的取数见 [nuxt.md](nuxt.md)，两者优先于本文。异步基础规则见 [concurrency.md](../languages/typescript/concurrency.md)，错误处理原则见 [error-handling.md](../general/error-handling.md)。

## 请求客户端

### 使用统一的请求客户端 【推荐】

在一处处理 baseURL、鉴权头、超时、HTTP 错误判定和错误转换。组件不直接调用 `fetch` 或 `axios`。响应按 [types.md](../languages/typescript/types.md) 的规则校验。204、202 和部分 201 响应没有响应体，这类接口的 schema 声明为空值。`Location`、`ETag`、`Retry-After` 从返回的 `headers` 读取。用 `{ code, message, data }` 包装的项目在这里解包。

**原因：** `fetch` 遇到 404、500 不会 reject，每处自己判断 `response.ok` 必有遗漏。

**示例：**

```ts
async function request<T>(
  path: string,
  init: RequestOptions,
  schema: z.ZodType<T>,
): Promise<{ data: T; headers: Headers }> {
  const timeout = AbortSignal.timeout(DEFAULT_TIMEOUT_MS);
  const signal = init.signal ? AbortSignal.any([init.signal, timeout]) : timeout;
  const response = await fetch(`${API_BASE}${path}`, { ...init, signal });
  if (!response.ok) throw await toApiError(response);
  const text = await response.text();
  const data = schema.parse(text === '' ? undefined : JSON.parse(text));
  return { data, headers: response.headers };
}
```

**来源：** MDN

### 接口函数按业务模块组织并声明类型 【推荐】

接口函数放在所属业务模块内，接收 `signal` 并声明返回类型，比如 `getOrder(id, signal)`。有 OpenAPI 文档时自动生成类型，避免前后端类型漂移。

### 设置超时并区分超时和取消 【必须】

`fetch` 没有超时选项，用 `AbortSignal.timeout()` 或客户端配置。超时的错误名是 `TimeoutError`，要提示用户；主动取消是 `AbortError`，不算失败，不提示。

**来源：** MDN

## 缓存与失效

### 缓存 key 包含所有影响结果的变量 【必须】

请求函数用到的变量都放进 key。TanStack Query 的 key 是可序列化的数组，元素顺序有意义，对象属性顺序无关。

**原因：** 变量不在 key 中时，不同参数会命中同一份缓存，显示错误数据。

**示例：**

```ts
useQuery({
  queryKey: ['orders', { status }],
  queryFn: ({ signal }) => listOrders(status, signal),
});
```

**来源：** TanStack Query 文档、SWR 文档

### 按数据变化频率设置缓存时间 【推荐】

TanStack Query 默认把缓存视为已过期，挂载、窗口聚焦、网络重连时都会后台刷新，无人使用的缓存 5 分钟后回收。SWR 也默认在聚焦和重连时重新验证。变化慢的数据设置 `staleTime`，不变的数据用 `useSWRImmutable`。

**来源：** TanStack Query 文档、SWR 文档

### 写操作后失效相关查询 【必须】

写操作成功后用 `invalidateQueries` 失效相关查询，或用接口返回值更新缓存。同一资源的 key 用相同前缀，`['orders']` 可以匹配该前缀下的所有查询。否则页面继续显示写之前的数据。

**来源：** TanStack Query 文档

## 竞态

### 处理请求竞态 【必须】

搜索框、切换 Tab 等场景下，旧请求可能晚于新请求返回并覆盖新结果。用请求库管理，并把 `signal` 传给请求，TanStack Query 只有用了 `signal` 才会真正取消。手写时用 `AbortController` 取消旧请求，每次请求新建一个。

**示例：**

```ts
useEffect(() => {
  const controller = new AbortController();
  search(keyword, controller.signal).then(setResults, ignoreAbort);
  return () => controller.abort();
}, [keyword]);
```

**来源：** MDN、TanStack Query 文档

## 重试

### 只自动重试幂等请求 【推荐】

POST、PATCH 不保证幂等，不自动重试，带 `Idempotency-Key` 的关键写操作除外。TanStack Query 的 mutation 默认不重试，不要全局打开。查询默认会自动重试，除 429 外的 4xx 重试没有意义，用 `retry` 函数排除。查询遇到 429 和 503 时，有 `Retry-After` 就按其等待时间重试。跨域时后端要暴露这个响应头，见 [web-risks.md](../backend/security/web-risks.md)。

**来源：** MDN、TanStack Query 文档

### 关键写操作的幂等键在重试时复用 【必须】

**适用：** 高风险

关键写操作在一次逻辑提交开始时生成 `Idempotency-Key`，由统一请求层发送。超时和网络错误后的手动重试复用同一个 key，用户修改内容后才换新 key。同 key 的请求仍在执行时后端返回 409，这时提示稍后再试。后端规则见 [rest.md](../backend/api/rest.md)。

**原因：** 超时后无法判断请求是否成功。重试时换新 key，后端会当作新请求，造成重复下单。

## 乐观更新

### 只对容易回滚的操作做乐观更新 【推荐】

点赞、勾选、排序适合，支付、下单等必须由后端确认的操作不做。只在一处展示时，用 mutation 的 `variables` 和 `isPending` 渲染临时项，不改缓存；多处都要感知时才改缓存。

**来源：** TanStack Query 文档

### 改缓存时完成取消、快照、回滚和失效 【必须】

四步缺一不可。SWR 用 `optimisticData` 配合 `rollbackOnError`，后者默认开启。

**原因：** 不取消进行中的请求，旧响应会覆盖乐观值；不回滚，失败后界面停留在错误状态；不失效，缓存与服务端不一致。

**示例：**

```ts
useMutation({
  mutationFn: updateTodo,
  onMutate: async (next) => {
    await queryClient.cancelQueries({ queryKey: todoKeys.detail(next.id) });
    const previous = queryClient.getQueryData<Todo>(todoKeys.detail(next.id));
    queryClient.setQueryData(todoKeys.detail(next.id), next);
    return { previous };
  },
  onError: (_error, next, context) => {
    queryClient.setQueryData(todoKeys.detail(next.id), context?.previous);
  },
  onSettled: (_data, _error, next) =>
    queryClient.invalidateQueries({ queryKey: todoKeys.detail(next.id) }),
});
```

**来源：** TanStack Query 文档、SWR 文档

## 上传下载

### `FormData` 上传不手动设置 `Content-Type` 【必须】

请求客户端的默认 JSON 请求头也不能应用到 `FormData`。手动设置后浏览器无法写入 boundary，服务端解析不了请求体。前端的类型和大小检查只为体验，后端规则见 [file-handling.md](../backend/security/file-handling.md)。

**来源：** MDN

### 需要上传进度时用 `XMLHttpRequest` 【推荐】

监听 `xhr.upload` 的 `progress` 事件，并把 `AbortSignal` 接到 `xhr.abort()`。跨域时注册监听器会触发预检请求。

**来源：** MDN、TanStack Query 文档

### Blob 下载后释放对象 URL 【推荐】

`URL.createObjectURL()` 创建的地址在下载触发后调用 `URL.revokeObjectURL()` 释放，否则造成内存泄漏。大文件优先让浏览器直接访问下载地址，不读入内存。

**来源：** MDN

## 错误分类

### 按错误类型分别处理 【推荐】

请求客户端把错误转换为统一类型，通用处理集中完成，业务错误交给调用方。路由层展示见 [routing.md](routing.md)，后端错误格式见 [rest.md](../backend/api/rest.md)。

| 错误 | 处理方式 |
|---|---|
| 400 | 通用提示，并上报错误监控 |
| 401 | 统一刷新凭证或跳转登录，并保留回跳地址；并发请求共用一次刷新 |
| 403 | 提示无权限 |
| 404 | 展示资源不存在 |
| 409 | 按错误标识区分，业务冲突展示原因，幂等请求执行中提示稍后再试 |
| 412 | 提示数据已变更，并重新加载 |
| 422 | 展示字段或业务错误，字段映射见 [forms.md](forms.md) |
| 429 | 查询按上面的重试规则处理，写操作提示稍后重试 |
| 5xx、网络错误 | 通用提示，提供重试入口，关键写操作复用幂等键 |

### 不向用户展示技术错误 【必须】

堆栈、SQL 错误不进入界面，错误信息要告诉用户下一步能做什么。技术细节上报到错误监控，比如 Sentry；上传 Source Map，但不公开部署，见 [observability.md](observability.md)。

## 字段约定

时刻和本地日期见 [i18n.md](i18n.md)，金额和 64 位整数的运算见 [idioms.md](../languages/typescript/idioms.md)，接口字段约定见 [rest.md](../backend/api/rest.md)。

环境变量见 [build-and-compat.md](build-and-compat.md)。
