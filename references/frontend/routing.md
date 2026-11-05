# 前端路由

本文规定客户端路由的设计、守卫、懒加载、滚动恢复和错误页。URL 查询参数见 [state-management.md](state-management.md)，Next.js 和 Nuxt 的文件路由见 [nextjs.md](nextjs.md)、[nuxt.md](nuxt.md)。

## 路由设计

### 路由表集中定义，用嵌套路由表达布局 【推荐】

路由配置放在一处。Vue Router 的权限和标题等信息写在 `meta` 上，通过声明 `RouteMeta` 补充类型。共享外壳放在布局路由，子页面通过 `<Outlet />` 或 `<RouterView />` 渲染。

**来源：** Vue Router 文档、React Router 文档

### 响应路由参数变化 【必须】

从 `/users/1` 切换到 `/users/2` 时组件实例被复用，只在挂载时执行的逻辑不会再运行，页面还显示旧数据。把参数放进缓存 key，或监听参数变化并取消旧请求。

**示例：**

```ts
watch(
  () => route.params.id,
  (id) => {
    const controller = new AbortController();
    onWatcherCleanup(() => controller.abort());
    loadUser(id, controller.signal).then((result) => {
      user.value = result;
    }, ignoreAbort);
  },
  { immediate: true },
);
```

**来源：** Vue Router 文档

## 守卫

### 登录和权限跳转集中在全局路由守卫 【推荐】

路由声明需要的条件，全局守卫统一判断和跳转，页面组件不各自处理。Vue Router 用 `beforeEach` 配合 `meta`。React Router 用挂在受保护父路由上的 `middleware`，声明式路由用受保护的布局路由。Next.js 和 Nuxt 的做法见 [nextjs.md](nextjs.md)、[nuxt.md](nuxt.md)。

**原因：** 判断散落在页面中，新页面容易漏掉，跳转规则也无法统一修改。

**示例：**

```ts
router.beforeEach((to) => {
  const auth = useAuthStore();
  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { name: 'Login', query: { redirect: to.fullPath } };
  }
});
```

```js
async function authMiddleware({ request }) {
  const user = await getUser();
  if (!user) {
    const { pathname, search } = new URL(request.url);
    throw redirect(`/login?redirect=${encodeURIComponent(pathname + search)}`);
  }
}
```

**来源：** Vue Router 文档、React Router 文档

### 守卫只影响体验，权限判断以后端为准 【必须】

守卫、隐藏菜单和隐藏按钮只决定用户看到什么。每个接口由后端校验权限，前端收到 401、403 时按 [data-fetching.md](data-fetching.md) 处理。

**原因：** 客户端逻辑很容易绕过，用户可以修改代码或直接调用接口。

**来源：** OWASP 授权速查表

### 守卫不产生循环或挂起 【必须】

重定向要排除目标页，登录页本身不能再触发跳转到登录页，否则形成无限重定向。Vue Router 守卫用返回值控制：`false` 取消，路由位置表示重定向，不返回值表示放行。不声明 `next` 参数，声明后路由会等待它被调用，导航可能挂起。

**来源：** Vue Router 文档

### 登录回跳地址只接受站内路径 【必须】

`redirect` 参数来自 URL，用户可以任意构造。用 URL 解析器解析后比较 origin，不用字符串前缀判断，`startsWith('/')` 挡不住以双斜杠开头的地址。

**原因：** 不校验时，登录后会被带到攻击者的站点。

**示例：**

```ts
function safeRedirectPath(raw: string | null): string {
  const origin = window.location.origin;
  if (raw === null || !URL.canParse(raw, origin)) return '/';
  const url = new URL(raw, origin);
  return url.origin === origin ? `${url.pathname}${url.search}${url.hash}` : '/';
}
```

**来源：** OWASP 未校验重定向速查表

## 懒加载

### 路由组件用动态 `import()` 懒加载 【推荐】

路由组件默认用动态 `import()`，打包工具会自动分割代码。页面很少时可以静态导入。Vue Router 把返回 Promise 的函数传给 `component`，不用 `defineAsyncComponent` 包裹路由组件。React Router 用路由的 `lazy` 字段；声明式路由用 `React.lazy`，在模块顶层声明并用 `Suspense` 包裹。

**原因：** 静态导入会把所有页面打进首屏包。

**示例：**

```ts
const vueRoutes = [{ path: '/orders', component: () => import('./pages/OrderList.vue') }];

const reactRoutes = [
  {
    path: '/orders',
    lazy: async () => {
      const { OrderList } = await import('./pages/OrderList');
      return { Component: OrderList };
    },
  },
];
```

**工具：** 用构建产物分析确认页面代码未进入入口包，见 [build-and-compat.md](build-and-compat.md)。

**来源：** Vue Router 文档、React Router 文档、React 文档

### 处理页面代码加载失败 【推荐】

**适用：** 线上服务

网络中断或发布后文件被清理时，动态导入会失败。React 中错误交给最近的错误边界，Vue Router 用 `router.onError` 处理，界面要提供刷新入口。否则点击链接没有反应，或者出现白屏。

**来源：** React 文档、Vue Router 文档

## 滚动恢复

### 后退恢复位置，新页面回到顶部 【必须】

单页应用切换路由不会重新加载页面，不处理时新页面停留在上一页的滚动位置。Vue Router 配置 `scrollBehavior`。React Router 在 Framework 和 Data 模式下渲染一个 `<ScrollRestoration />`。

**示例：**

```ts
const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, _from, savedPosition) {
    if (savedPosition) return savedPosition;
    if (to.hash) return { el: to.hash, top: HEADER_HEIGHT };
    return { top: 0 };
  },
});
```

**来源：** Vue Router 文档、React Router 文档

### 页内切换不重置滚动，滚动只由一方管理 【推荐】

只改变查询参数的 Tab 和筛选不回到顶部，React Router 用 `preventScrollReset`。路由库接管后，不再手写 `window.scrollTo` 或修改 `history.scrollRestoration`。

**来源：** React Router 文档、Vue Router 文档、MDN

## 404 和权限页

### 配置兜底的 404 路由 【必须】

Vue Router 用 `/:pathMatch(.*)*`，React Router 用 `path: '*'`，放在布局路由内可保留外壳。使用 History 模式时，服务器要把未知路径回退到入口页面。

**来源：** Vue Router 文档、React Router 文档

### 区分路由不存在、资源不存在和无权限 【推荐】

错误页在当前 URL 就地渲染，不跳转到 `/404`。

| 情况 | 处理 |
|---|---|
| URL 没有匹配的路由 | 404 页 |
| 路由存在，接口返回 404 | 在当前路由展示资源不存在 |
| 接口返回 403 | 权限页，不跳转登录 |
| 接口返回 401 | 跳转登录并保留回跳地址 |

**原因：** 跳转后 URL 丢失，用户无法刷新重试，也无法反馈出错地址。

### 配置根错误边界 【推荐】

React Router 至少导出一个根 `ErrorBoundary`，分别处理响应错误、`Error` 实例和其他抛出值，生产环境不展示堆栈。错误边界用于未预期的错误，不用抛错做普通流程控制。例外是资源不存在，loader 中用 `throw data(message, { status: 404 })`。表单校验错误见 [forms.md](forms.md)。

**来源：** React Router 文档

### 导航后更新页面标题 【推荐】

在全局后置钩子中读取路由的标题信息，比如 Vue Router 的 `afterEach`。路由切换后的焦点管理见 [accessibility.md](accessibility.md)。

**来源：** Vue Router 文档
