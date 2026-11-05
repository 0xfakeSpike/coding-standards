# Next.js

本文针对 App Router，先阅读 [react.md](react.md)。缓存规则以开启 `cacheComponents` 为前提。服务端取数和错误处理以本文为准，优先于 [data-fetching.md](data-fetching.md) 和 react.md 中面向客户端渲染的规则。

## App Router

### 用文件约定表达路由状态 【推荐】

`layout` 写共享布局，`page` 写公开页面，`loading` 写加载界面，`error` 和 `global-error` 处理未捕获异常，`not-found` 写 404，`route` 写接口。路由内部文件放在 `_components` 这类私有目录，用 `(group)` 分组路由而不影响 URL。

**来源：** Next.js 文档

### `params` 和 `searchParams` 要 `await`，并按用户输入处理 【必须】

Next.js 15 及以上它们是 Promise。它们也是可以被任意修改的外部输入，使用前要校验，不能用来判断权限。

```tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return <Order id={id} />;
}
```

**来源：** Next.js 文档

### 区分预期错误和未捕获异常 【推荐】

预期错误作为返回值处理，配合 `useActionState` 展示，资源不存在时调用 `notFound()`。程序缺陷直接抛出，由 `error.tsx` 捕获，该文件必须是 Client Component。表单细则见 [forms.md](forms.md)。

**来源：** Next.js 文档

### `proxy.ts` 用 `matcher` 收窄范围 【推荐】

`proxy.ts` 需要 Next.js 16 及以上。不写 `matcher` 时，它会拦截包括静态资源在内的所有请求。它不能作为唯一的鉴权手段。

**来源：** Next.js 文档

## Server 与 Client 组件

### 默认用 Server Component，把 `'use client'` 放到叶子 【推荐】

只有需要 State、事件、Effect 或浏览器 API 的组件才标记。比如布局里只有搜索框需要交互时，只标记 `Search` 所在文件，不标记整个 `Layout`。

**原因：** 被标记文件的所有导入都会进入客户端产物。

**来源：** Next.js 文档

### 用 `children` 把 Server 组件放进 Client 组件 【推荐】

不要在 Client 组件里导入 Server 组件，而是通过 `children` 或其他 Prop 接收，比如 `<Modal><Cart /></Modal>`。Context Provider 也包成接收 `children` 的 Client 组件，放到尽量深的位置。

**来源：** Next.js 文档

### 传给 Client 组件的 Props 可序列化且最小 【必须】

只传渲染需要的字段，不传数据库记录或完整用户对象。

**示例：**

反例：

```tsx
<Profile user={userRow} />
```

正例：

```tsx
<Profile name={userRow.name} avatarUrl={userRow.avatarUrl} />
```

**原因：** 传给 Client 组件的所有字段都会发送到浏览器。

**来源：** Next.js 文档

### 服务端专用模块导入 `server-only` 【必须】

含机密或数据库访问的模块在文件开头写 `import 'server-only'`。

**原因：** 被客户端误导入时，构建会直接报错。

**来源：** Next.js 文档

## 数据获取与缓存

通用请求规则见 [data-fetching.md](data-fetching.md)。

### 在 Server Component 中靠近使用处取数 【推荐】

相同的 `fetch` 请求会自动去重，可以在需要数据的组件里直接请求。传了 `signal` 的 `fetch` 不参与去重，需要超时的请求和 ORM、数据库查询一样用 `React.cache` 包装。无依赖的请求用 `Promise.all` 并行发起，不要连续 `await`。

**来源：** Next.js 文档

### 慢数据和请求时数据放进 `<Suspense>` 【推荐】

读取 `cookies()`、`headers()`、`searchParams` 或未缓存数据的组件，用靠近它的 `<Suspense>` 包住，其余内容先输出。不要在 `layout` 中直接读取这些数据，同级的 `loading` 兜不住。

**来源：** Next.js 文档

### 新项目用数据访问层 【推荐】

建立 `server-only` 的数据访问层，在其中做权限检查，只返回最小的数据对象。一个项目只选一种取数方式。

**原因：** 集中后权限检查一致，安全审计也有明确入口。

**来源：** Next.js 文档

### 渲染过程不做数据变更 【必须】

登出、写库、失效缓存都放在 Server Action 或 Route Handler 中。

**来源：** Next.js 文档

### `use cache` 配 `cacheLife` 写明时效 【推荐】

不写时用隐式的 `default` 配置，调用处看不出缓存多久。需要每次请求都最新的数据不要缓存。

```tsx
export async function getProducts() {
  'use cache';
  cacheLife('hours');
  cacheTag('products');
  return db.query('SELECT * FROM products');
}
```

**来源：** Next.js 文档

### 缓存作用域内不读请求数据 【必须】

`cookies()`、`headers()`、`searchParams` 不能在 `use cache` 内读取，包括它调用的函数。在外部读取后作为参数传入，参数会成为缓存键的一部分。

**原因：** 这类错误可能通过构建，到运行时才失败。

**来源：** Next.js 文档

### 按场景选择缓存更新方式 【推荐】

优先按标签失效，比按路径更精确。`updateTag` 和 `refresh` 需要 Next.js 16 及以上。

| 方法 | 场景 |
|---|---|
| `updateTag` | 用户要立刻看到自己的修改，只能在 Server Action 中用 |
| `revalidateTag(tag, 'max')` | 允许短暂旧数据，后台刷新 |
| `revalidatePath` | 只影响一个路由，不值得打标签 |
| `refresh` | 不失效缓存，只刷新当前路由 |

**来源：** Next.js 文档

## Server Actions

### 每个 Action 内部校验身份和权限 【必须】

Action 可以被直接 POST 调用。页面级检查、`proxy.ts` 和只渲染表单都不算安全边界。除了登录状态，还要检查用户对该资源的权限。

```ts
export async function completeItem(input: unknown): Promise<ActionResult> {
  const session = await auth();
  if (!session?.user) return { status: 'unauthenticated' };
  const parsed = itemIdSchema.safeParse(input);
  if (!parsed.success) return { status: 'invalid' };
  const { count } = await db.item.updateMany({
    where: { id: parsed.data, ownerId: session.user.id },
    data: { completed: true },
  });
  return count === 1 ? { status: 'ok' } : { status: 'not-found' };
}
```

**来源：** Next.js 文档

### 校验入参并控制返回值 【必须】

`FormData` 等入参都不可信，客户端只传 ID 和用户的修改，其余数据从可信来源重新读取。返回值只包含界面需要的字段。

**原因：** 返回值会序列化发送到客户端，Schema 校验也不能证明资源归属。

**来源：** Next.js 文档

### Action 只用于变更 【推荐】

读数据放在 Server Component。客户端按顺序逐个派发 Action，用 `Promise.all` 也不会并行，需要并行时放在单个 Action 内完成。

**来源：** Next.js 文档

### 先刷新缓存再 `redirect` 【必须】

`redirect` 通过抛出异常中断执行，后面的代码不会运行。

**来源：** Next.js 文档

## 环境变量

前端环境变量的要求见 [build-and-compat.md](build-and-compat.md)。不带 `NEXT_PUBLIC_` 前缀的变量只在服务端可用，机密只在数据访问层读取。

### 公开变量按构建时常量使用 【必须】

`NEXT_PUBLIC_` 变量在构建时内联，之后修改不生效。要写完整的 `process.env.NEXT_PUBLIC_ANALYTICS_ID`，用变量名动态读取不会被替换。同一镜像部署到多个环境时，在服务端读取运行时变量。

**来源：** Next.js 文档

### `.env` 文件不进版本库 【必须】

**例外：** 不含机密的 `.env.test` 可以提交，`.env*.local` 不提交。

**来源：** Next.js 文档
