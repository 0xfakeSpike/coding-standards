# Vue 3

先阅读 [components.md](components.md) 中的框架无关规则。本文多数模板和组件规则可用 `eslint-plugin-vue` 检查。

## 基本写法

### 新代码用 `<script setup>` 和 TypeScript 【推荐】

使用 Composition API。SFC 块顺序统一为 `<script>`、`<template>`、`<style>`。样式方案见 [css.md](css.md)，用 `scoped` 时选择器用类名。

**来源：** Vue 风格指南

### 组件名用多单词 【推荐】

比如 `UserCard`，不用 `Card`，根组件 `App` 除外。

**原因：** Vue 模板里组件可以写成 kebab-case，单词名可能和 HTML 元素冲突。

**来源：** Vue 风格指南

## 响应式

### 基础状态优先用 `ref` 【推荐】

`reactive` 只支持对象，不能整体替换，解构后也会失去响应性。

**来源：** Vue 文档

### 防止响应式丢失 【必须】

解构 `reactive` 对象时用 `toRefs` 或 `toRef`。解构 Props 保持响应性需要 Vue 3.5 及以上，传给 `watch` 或 Composable 时要包成 getter，写 `watch(() => foo, reload)`，不写 `watch(foo, reload)`。

**来源：** Vue 文档

### 派生数据用 `computed`，`watch` 只做副作用 【推荐】

`watch` 用于发请求、写存储这类副作用，不用它同步另一份状态。

**来源：** Vue 文档

### `computed` 保持纯计算 【必须】

不发请求、不修改状态、不操作 DOM。`sort`、`reverse` 这类方法要先复制数组，写 `[...items.value].sort(byPrice)`。

**工具：** `vue/no-side-effects-in-computed-properties`。

**来源：** Vue 文档

### 修改状态后读取 DOM 先等待更新 【必须】

修改状态后要读取 DOM 时，先 `await nextTick()`。DOM 更新是异步批量执行的，立即读取拿到的是旧值。

### 清理副作用 【必须】

定时器和事件监听在 `onUnmounted` 中清理。`watch` 中的请求用 `onCleanup` 或 `onWatcherCleanup` 取消，后者需要 Vue 3.5 及以上，异步回调里创建的 watcher 要手工停止。

**原因：** 不清理会造成内存泄漏，旧请求也可能覆盖新结果。

**来源：** Vue 文档

## Composables

### 以 `use` 开头，入参接受 ref，返回 ref 对象 【推荐】

返回由 `ref` 组成的普通对象，调用方解构时仍能保持响应性。入参接受 ref 或 getter，用 `toValue()` 读取。在 `watchEffect` 内调用 `toValue()`，依赖才会被追踪。

**来源：** Vue 文档

### 在 `setup` 中同步调用 【必须】

Composable 要在 `<script setup>` 或 `setup()` 中同步调用，不放进异步回调。访问 DOM 的副作用放进 `onMounted`。

**原因：** Vue 要靠当前组件实例注册生命周期钩子和 watcher，服务端渲染时也没有 DOM。

**来源：** Vue 文档

### 纯逻辑复用用 Composable 【推荐】

不用无渲染组件。既要封装逻辑又要组合界面时，才用作用域插槽。

**来源：** Vue 文档

## 组件通信

### Props 声明类型，事件用 `defineEmits` 声明 【推荐】

```vue
<script setup lang="ts">
const { title, size = 'sm' } = defineProps<{ title: string; size?: 'sm' | 'lg' }>();
const emit = defineEmits<{ submit: [id: string] }>();
</script>
```

**来源：** Vue 风格指南、Vue 文档

### 双向绑定用 `v-model` 或 `defineModel` 【推荐】

不修改 Props 的要求见 [components.md](components.md)，也不写 `$parent`。`defineModel` 设置 `default` 时，父组件不传值会导致两端不一致。

**工具：** `vue/no-mutating-props`。

**来源：** Vue 风格指南、Vue 文档

### `provide` 和 `inject` 用 `InjectionKey` 声明类型 【推荐】

Key 放在单独文件里。提供方暴露只读状态和更新函数，不让注入方直接修改。

```ts
export const counterKey: InjectionKey<{
  count: Readonly<Ref<number>>;
  increment: () => void;
}> = Symbol('counter');

const count = ref(0);
provide(counterKey, { count: readonly(count), increment: () => count.value++ });
```

**原因：** 状态和修改集中在提供方，数据流才能追踪。

**来源：** Vue 文档

## 模板规则

### `v-for` 配 `:key`，不和 `v-if` 用在同一元素上 【必须】

过滤列表用 `computed`，写 `v-for="user in activeUsers"`。隐藏整个列表时把 `v-if` 放到外层元素。

**原因：** `v-if` 优先级更高，读不到 `v-for` 的变量。

**工具：** `vue/require-v-for-key`、`vue/no-use-v-if-with-v-for`。

**来源：** Vue 风格指南

### 模板保持简单 【推荐】

复杂表达式移到 `computed` 或函数，复杂的 `computed` 拆成多个命名清楚的小 `computed`。

**来源：** Vue 风格指南

`v-html` 和用户内容的要求见 [security.md](security.md)。用户提供的内容也不能作为组件模板。

## Pinia

状态放在哪一层见 [state-management.md](state-management.md)。

### Store 命名统一 【推荐】

函数命名为 `useXxxStore`，每个 Store 一个文件。

**来源：** Pinia 文档

### Setup Store 返回全部状态 【必须】

Store 的 `id` 全局唯一。Pinia 不支持私有状态，Setup Store 也不要返回只读状态。

**原因：** 少返回状态时，SSR、开发工具和插件都会出错。

**来源：** Pinia 文档

### 解构 Store 用 `storeToRefs` 【必须】

状态和 getter 写 `const { name } = storeToRefs(store)`，Action 可以直接解构。

**原因：** 直接解构状态和 getter 会失去响应性。

**来源：** Pinia 文档

### 注意 Store 的调用时机 【必须】

- 不在模块顶层调用 `useStore()`，放进 `app.use(pinia)` 之后才运行的函数。
- SSR 时在组件外调用，要显式传入 `pinia` 实例。
- 异步 Action 中，所有 `useStore()` 写在第一个 `await` 之前。
- 互相依赖的 Store 不在 setup 中互读状态，把读取放进 getter 或 Action。

**原因：** 否则调用会失败或无限循环，SSR 时还可能让状态在请求之间泄漏。

**来源：** Pinia 文档

## 路由

路由懒加载和守卫见 [routing.md](routing.md)。

## 性能

通用性能规则见 [performance.md](performance.md)。

### 传给子组件的 Props 保持稳定 【推荐】

在父组件算好结果再传入，写 `:active="item.id === activeId"`，不把 `activeId` 传给每一项。这样只有受影响的子组件才更新。

**来源：** Vue 文档

### 按数据规模优化 【推荐】

大型数据用 `shallowRef`，通过替换根值触发更新。长列表虚拟化，非首屏组件用 `defineAsyncComponent`。内容不再变化时用 `v-once`，大型子树确认有更新瓶颈时用 `v-memo`。

**来源：** Vue 文档
