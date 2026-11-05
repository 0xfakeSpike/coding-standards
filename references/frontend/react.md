# React

先阅读 [components.md](components.md) 中的框架无关规则。状态分层见 [state-management.md](state-management.md)，请求见 [data-fetching.md](data-fetching.md)。

## 渲染规则

### 组件和 Hook 保持纯 【必须】

相同的 Props、State 和 Context 要得到相同的输出。渲染期间不修改外部变量，不发请求，不读写 `ref.current`。Server Component 的取数除外，见 [nextjs.md](nextjs.md)。

**原因：** React 可能多次渲染同一组件，副作用会重复执行，结果也不可预测。

**工具：** `react-hooks/purity`、`react-hooks/globals`、`react-hooks/refs`。

**来源：** React 文档

### Props 和 State 不可变 【必须】

不要直接修改它们，也不要修改传给 Hook 或 JSX 之后的值。比如写 `setItems([...items, newItem])`，不写 `items.push(newItem)`。

**原因：** 引用不变时 React 可能检测不到变化。

**工具：** `react-hooks/immutability`。

**来源：** React 文档

### 不在组件内部定义组件 【必须】

组件函数定义在模块顶层。

**原因：** 每次渲染都会生成新的组件类型，子树被卸载重建并丢失状态。

**工具：** `react-hooks/static-components`。

**来源：** React 文档

### 用 `key` 重置状态 【推荐】

切换到另一个实体时要清空全部状态，写 `<Chat key={contact.id} contact={contact} />`，不用 Effect 手工清空。

**来源：** React 文档

## Hooks

### 开启 `eslint-plugin-react-hooks`，Hook 只在顶层调用 【必须】

使用 `recommended` 预设。Hook 只在组件或自定义 Hook 的顶层调用，不放进条件、循环和嵌套函数，也不放在提前返回之后。自定义 Hook 以 `use` 开头。

**工具：** `react-hooks/rules-of-hooks`。

**来源：** React 文档

### 外部数据源用 `useSyncExternalStore` 【推荐】

订阅第三方 Store 或 `navigator.onLine` 这类浏览器状态时使用，不在 Effect 里手工同步到 State。`getSnapshot` 要返回缓存的不可变值，`subscribe` 要保持稳定，服务端渲染时提供 `getServerSnapshot`。

**来源：** React 文档

## Effect

### `useEffect` 只用于和外部系统同步 【必须】

外部系统指订阅、定时器、DOM 操作、第三方组件和网络连接。没有外部系统时，按下表处理。

| 场景 | 做法 |
|---|---|
| 从 Props 或 State 派生数据 | 渲染时计算 |
| 用户操作引起的逻辑和请求 | 写在事件处理函数里 |
| Prop 变化时重置全部状态 | 传 `key` |
| 通知父组件状态变化 | 在同一事件里同时更新 |
| 应用启动时的一次性初始化 | 放在模块顶层或入口 |

**示例：**

反例：

```tsx
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);
```

正例：

```tsx
const fullName = `${firstName} ${lastName}`;
```

**原因：** 多余的 Effect 带来额外渲染，连锁的 Effect 也很难追踪。

**工具：** `react-hooks/set-state-in-effect`。

**来源：** React 文档

### 依赖数组如实填写 【必须】

不用忽略注释绕过 lint。依赖变化过于频繁时调整代码结构，比如把对象移进 Effect。确实不该触发重新同步的逻辑，放进 `useEffectEvent`，需要 React 19.2 及以上。

**原因：** 依赖不完整会读到旧闭包。

**工具：** `react-hooks/exhaustive-deps`。

**来源：** React 文档

### Effect 启动的工作要清理 【必须】

订阅、定时器、连接和请求都要在清理函数里停止或忽略结果，防止旧响应覆盖新结果。Strict Mode 在开发环境多执行一轮挂载和卸载，就是为了暴露漏写的清理，不要用 ref 绕开。

```tsx
useEffect(() => {
  const controller = new AbortController();
  loadUser(id, controller.signal).then(setUser, ignoreAbort);
  return () => controller.abort();
}, [id]);
```

**原因：** 不清理会造成泄漏和竞态。

**来源：** React 文档

## 性能

### 不默认包 `useMemo`、`useCallback`、`memo` 【推荐】

先用 React DevTools Profiler 或生产构建测量，再优先调整结构：把 JSX 作为 `children` 传入，把状态放到尽量低的层级，去掉多余的 Effect。之后仍有瓶颈才加缓存。

**原因：** 不必要的缓存只会让代码更难读。

**来源：** React 文档

### 正确性不依赖缓存 【必须】

删掉 `useMemo` 和 `memo` 后，代码仍要行为正确。需要稳定身份时用 State 或 Ref。

**原因：** React 在特定情况下会丢弃缓存。

**来源：** React 文档

### 非紧急更新用 Transition 【推荐】

切换标签页、路由这类可能较慢的更新放进 `startTransition`，界面保持可响应。受控输入框的值不能放进 Transition。

**来源：** React 文档

## Suspense

### 按期望的加载顺序放置边界 【推荐】

一起出现的内容放在同一个边界，能分批出现的内容嵌套边界，不给每个组件都包一层。更新时用 `startTransition` 或 `useDeferredValue` 保留已展示的内容，不让 fallback 替换它。

**来源：** React 文档

### 只有支持 Suspense 的数据源才触发边界 【必须】

`lazy`、`use` 读取的 Promise 和支持 Suspense 的框架会触发。Effect 和事件处理函数里的请求不会触发，要自己处理加载状态。传给 `use` 的 Promise 要缓存，不能每次渲染新建。

**来源：** React 文档

## 错误边界

### 用错误边界隔离局部错误 【推荐】

错误边界用类组件或 `react-error-boundary` 实现。在页面、面板这类有意义的区域放置边界，防止局部报错导致白屏，并在 `componentDidCatch` 中上报，见 [observability.md](observability.md)。

**来源：** React 文档

### 错误边界之外的错误手工处理 【必须】

错误边界不捕获事件处理函数、异步回调和服务端渲染中的错误。这些地方要自己捕获，并更新界面告知用户。Next.js 的 `error.tsx` 能接住 Server Component 的错误，见 [nextjs.md](nextjs.md)。

**来源：** React 文档

## React Compiler

### 启用后新代码不手写缓存 【推荐】

由编译器处理缓存，只在需要精确控制时使用 `useMemo` 和 `useCallback`，比如作为 Effect 依赖的值。已有的手写缓存先保留，删除前要测试。

**原因：** 删除手写缓存可能改变编译结果。

**来源：** React 文档

### 修复诊断问题，不长期跳过 【推荐】

编译器会跳过有诊断问题的组件，其余部分照常编译。按 ESLint 提示逐步修复，`"use no memo"` 只用于临时排查。

**工具：** `eslint-plugin-react-hooks`。

**来源：** React 文档
