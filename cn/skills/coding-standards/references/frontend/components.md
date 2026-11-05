# 前端组件

本文规则不依赖具体框架，框架细则见 [react.md](react.md) 和 [vue.md](vue.md)。目录和模块边界见 [architecture.md](architecture.md)。

## 组件分类

### 按职责区分组件 【推荐】

| 类别 | 职责 | 能否请求数据 | 位置 |
|---|---|---|---|
| 基础组件 | 按钮、输入框等无业务含义的 UI | 否 | `components/` 或设计系统 |
| 布局组件 | 页面框架和区域排布 | 否 | `components/` |
| 业务组件 | 展示和操作某个业务对象 | 按需 | `features/` |
| 页面组件 | 组装业务组件，对接路由 | 是 | 路由入口 |

### 一个组件一个职责 【推荐】

出现下面的信号时，考虑拆分或改用组合：

- 描述组件时需要用“并且”。
- Props 里混着几组互不相关的配置。
- 多个布尔 Prop 在切换组件的不同形态。
- 改一种使用场景时经常影响另一种。
- 同一个组件同时处理请求、业务判断和展示。
- 其中一段 UI 有独立状态，也能单独命名。

**原因：** 职责混杂的组件难以复用和测试，修改风险也高。

### 通用组件不依赖业务，业务逻辑放到组件外 【推荐】

通用组件的数据通过 Props 传入，不直接调用业务接口。业务逻辑放进 Hook、Composable 或普通函数，组件专注渲染，这样逻辑才便于复用和测试。

## 命名

### 组件名用 PascalCase 【推荐】

比如 `UserCard`。回调属性用 `on` 开头，内部处理函数用 `handle` 开头，比如 `onSubmit` 和 `handleSubmit`。Vue 的额外要求见 [vue.md](vue.md)。

## 同类组件共用一套

### 同类组件共用一套 【必须】

两处都有同类阅读器时，共用同一套组件，差异通过 Props 或插槽表达。新增组件前先查找已有实现，能扩展就不另写一份。

**示例：**

反例：

```text
features/contract/components/ContractReader.tsx
features/report/components/ReportReader.tsx
```

正例：

```tsx
<DocumentReader source={contract.fileUrl} toolbar={<ContractActions />} />
<DocumentReader source={report.fileUrl} toolbar={<ReportActions />} />
```

**原因：** 多份实现迟早行为不一致，修复一处时也容易漏掉另一处。

**例外：** 只是外观相似，但业务含义和变化原因不同。

**来源：** 团队

## Props 设计

### 只传组件需要的数据 【推荐】

Props 只包含组件实际用到的字段，不把整个业务对象传下去。

**原因：** 依赖越小越容易复用，也越不容易把敏感字段传到不该去的地方。

### 互斥形态用一个枚举 Prop 【推荐】

写 `<Button variant="danger" />`，不写 `<Button primary danger ghost />`。类型写法见 [types.md](../languages/typescript/types.md)。

**原因：** 多个布尔值能组合出不存在的状态。

### 子组件不修改传入的数据 【必须】

数据向下传，变化通过回调或事件向上通知。对象和数组 Prop 的内部字段也不能改。

**原因：** 单向数据流才能追踪状态是在哪里变化的。

**来源：** React 文档、Vue 风格指南

### 不把 Props 复制进状态 【必须】

直接使用 Prop，或在渲染时从 Prop 计算。

**原因：** 复制后只有初始值生效，之后父组件的更新会被忽略。

**例外：** 确实只要初始值时，用 `initialValue`、`defaultValue` 这类名字表明意图。

**来源：** React 文档、Vue 文档

## 组合模式

### 用组合解决属性透传 【推荐】

中间层只负责转发属性时，优先用 `children` 或 `slot` 让数据的持有者直接渲染使用方。组合不够用时，才用 Context 或 `provide`/`inject`。

**示例：**

反例：

```tsx
<Layout user={user} posts={posts} />
```

正例：

```tsx
<Layout header={<UserMenu user={user} />}>
  <PostList posts={posts} />
</Layout>
```

**来源：** React 文档

### 用子组件代替成组的配置 Prop 【推荐】

某个区域的内容和结构经常变化时，提供 `CardHeader`、`CardFooter` 这样的子组件或插槽，不继续增加 `showIcon`、`footerText` 这类配置 Prop。

## 受控与非受控

### 共享状态提升到共同父组件 【推荐】

多个组件需要同一份状态时，提升到最近的共同父组件，再向下传递。权威来源的要求见 [state-management.md](state-management.md)。

**来源：** React 文档

### 需要协调时用受控组件 【推荐】

父组件要读取、校验或联动某个值时用受控方式，状态只和组件自身有关时用非受控方式。两种都支持时分开命名：受控用 `value` 配 `onChange`，非受控用 `defaultValue`，Vue 组件用 `v-model`。表单控件的取舍见 [forms.md](forms.md)。

**来源：** React 文档

## UI 状态

### 处理完整的 UI 状态 【必须】

展示异步数据的组件要处理加载中、错误、成功状态，列表还要处理空数据。框架和请求库已提供 Suspense、`loading` 文件或 `status` 时直接使用。手写状态时用一个可辨识联合字段，不用多个布尔值。错误界面说明发生了什么，并提供重试或返回入口。

```tsx
switch (state.status) {
  case 'loading':
    return <Skeleton />;
  case 'error':
    return <ErrorPanel error={state.error} onRetry={reload} />;
  case 'empty':
    return <EmptyState />;
  case 'success':
    return <OrderTable orders={state.orders} />;
  default: {
    const unhandled: never = state;
    throw new Error(`未处理的状态 ${JSON.stringify(unhandled)}`);
  }
}
```

**原因：** 只实现成功状态是最常见的遗漏，用户会看到白屏或空表格。

**来源：** React 文档

## 列表和派生数据

### 列表 `key` 用稳定的业务 ID 【必须】

不用数组下标或随机数。本地创建的数据在创建时分配 ID，不在渲染时生成。

**原因：** 增删和排序时，不稳定的 `key` 会造成状态错位或组件反复重建。

**例外：** 永远不会增删和排序的静态列表可以用下标。

**来源：** React 文档、Vue 风格指南

能计算得到的值不存成状态，见 [state-management.md](state-management.md)。

## 组件文档

### 共享组件在 Storybook 或项目文档中说明 【推荐】

**适用：** 多人协作

基础组件和跨功能组件要有文档，写明用途、Props、事件、插槽和使用限制。团队已用 Storybook 时，用 `autodocs` 标签生成文档页，否则写进项目文档。

**原因：** 使用方先查文档才能发现已有组件，避免重复实现。

**来源：** Storybook 文档

### 每种状态写一个 Story 【推荐】

Story 与组件文件放在一起，每个具名导出对应一种状态，覆盖主要变体以及加载、空、错误状态。Story 文件不进生产包。

**来源：** Storybook 文档
