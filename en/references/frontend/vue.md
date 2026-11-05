# Vue 3

Read the framework-agnostic rules in [components.md](components.md) first. Most template and component rules in this document can be checked with `eslint-plugin-vue`.

## Basics

### Use `<script setup>` and TypeScript in new code [Recommended]

Use the Composition API. Order SFC blocks as `<script>`, `<template>`, `<style>`. See [css.md](css.md) for styling approaches. With `scoped`, use class names in selectors.

**Source:** Vue Style Guide

### Use multi-word component names [Recommended]

For example `UserCard`, not `Card`. The root component `App` is excepted.

**Why:** Vue templates can write components in kebab-case, so single-word names may clash with HTML elements.

**Source:** Vue Style Guide

## Reactivity

### Prefer `ref` for basic state [Recommended]

`reactive` supports only objects, cannot be replaced as a whole, and loses reactivity when destructured.

**Source:** Vue Docs

### Prevent loss of reactivity [Required]

Use `toRefs` or `toRef` when destructuring a `reactive` object. Keeping destructured Props reactive requires Vue 3.5 or later. When passing them to `watch` or a Composable, wrap them in a getter: write `watch(() => foo, reload)`, not `watch(foo, reload)`.

**Source:** Vue Docs

### Use `computed` for derived data and `watch` only for side effects [Recommended]

Use `watch` for side effects such as sending requests and writing to storage. Do not use it to sync another copy of state.

**Source:** Vue Docs

### Keep `computed` pure [Required]

Do not send requests, modify state, or manipulate the DOM. Copy the array before methods such as `sort` and `reverse`: write `[...items.value].sort(byPrice)`.

**Tooling:** `vue/no-side-effects-in-computed-properties`.

**Source:** Vue Docs

### Wait for updates before reading the DOM after changing state [Required]

When you need to read the DOM after changing state, `await nextTick()` first. DOM updates are batched asynchronously, so an immediate read gets the old value.

### Clean up side effects [Required]

Clean up timers and event listeners in `onUnmounted`. Cancel requests in `watch` with `onCleanup` or `onWatcherCleanup`, the latter of which requires Vue 3.5 or later. Stop watchers created in async callbacks manually.

**Why:** Missing cleanup causes memory leaks, and old requests may overwrite new results.

**Source:** Vue Docs

## Composables

### Start with `use`, accept refs, and return an object of refs [Recommended]

Return a plain object made of `ref`s, so callers keep reactivity when destructuring. Accept refs or getters as arguments and read them with `toValue()`. Call `toValue()` inside `watchEffect` so that dependencies are tracked.

**Source:** Vue Docs

### Call synchronously in `setup` [Required]

Call Composables synchronously in `<script setup>` or `setup()`, not in async callbacks. Put side effects that access the DOM in `onMounted`.

**Why:** Vue relies on the current component instance to register lifecycle hooks and watchers, and there is no DOM during server-side rendering.

**Source:** Vue Docs

### Use Composables for pure logic reuse [Recommended]

Do not use renderless components. Use scoped slots only when you need both to encapsulate logic and to compose UI.

**Source:** Vue Docs

## Component communication

### Type Props and declare events with `defineEmits` [Recommended]

```vue
<script setup lang="ts">
const { title, size = 'sm' } = defineProps<{ title: string; size?: 'sm' | 'lg' }>();
const emit = defineEmits<{ submit: [id: string] }>();
</script>
```

**Source:** Vue Style Guide, Vue Docs

### Use `v-model` or `defineModel` for two-way binding [Recommended]

See [components.md](components.md) for the rule against modifying Props. Do not write `$parent` either. When `defineModel` sets `default`, a parent that passes no value leaves the two sides out of sync.

**Tooling:** `vue/no-mutating-props`.

**Source:** Vue Style Guide, Vue Docs

### Type `provide` and `inject` with `InjectionKey` [Recommended]

Keep keys in a separate file. The provider exposes read-only state and update functions, and does not let injectors modify state directly.

```ts
export const counterKey: InjectionKey<{
  count: Readonly<Ref<number>>;
  increment: () => void;
}> = Symbol('counter');

const count = ref(0);
provide(counterKey, { count: readonly(count), increment: () => count.value++ });
```

**Why:** Data flow is traceable only when state and changes are centralized in the provider.

**Source:** Vue Docs

## Template rules

### Pair `v-for` with `:key` and keep it off the same element as `v-if` [Required]

Filter lists with `computed`: write `v-for="user in activeUsers"`. To hide the whole list, put `v-if` on the outer element.

**Why:** `v-if` has higher priority and cannot read `v-for` variables.

**Tooling:** `vue/require-v-for-key`, `vue/no-use-v-if-with-v-for`.

**Source:** Vue Style Guide

### Keep templates simple [Recommended]

Move complex expressions to `computed` or functions, and split complex `computed` into several small, clearly named `computed`.

**Source:** Vue Style Guide

See [security.md](security.md) for `v-html` and user content requirements. User-provided content must not be used as a component template either.

## Pinia

See [state-management.md](state-management.md) for which layer state belongs in.

### Name Stores consistently [Recommended]

Name the function `useXxxStore`, with one Store per file.

**Source:** Pinia Docs

### Return all state from Setup Stores [Required]

Store `id`s are globally unique. Pinia does not support private state, and Setup Stores must not return read-only state either.

**Why:** Omitting state breaks SSR, devtools, and plugins.

**Source:** Pinia Docs

### Destructure Stores with `storeToRefs` [Required]

For state and getters, write `const { name } = storeToRefs(store)`. Actions can be destructured directly.

**Why:** Destructuring state and getters directly loses reactivity.

**Source:** Pinia Docs

### Watch when Stores are called [Required]

- Do not call `useStore()` at the module top level. Put it in a function that runs after `app.use(pinia)`.
- In SSR, pass the `pinia` instance explicitly when calling outside a component.
- In async Actions, write all `useStore()` calls before the first `await`.
- Stores that depend on each other do not read each other's state in setup. Move those reads into getters or Actions.

**Why:** Otherwise calls fail or loop infinitely, and in SSR state may leak between requests.

**Source:** Pinia Docs

## Routing

See [routing.md](routing.md) for route lazy loading and guards.

## Performance

See [performance.md](performance.md) for general performance rules.

### Keep Props passed to child components stable [Recommended]

Compute results in the parent before passing them: write `:active="item.id === activeId"`, rather than passing `activeId` to every item. Then only affected child components update.

**Source:** Vue Docs

### Optimize by data size [Recommended]

Use `shallowRef` for large data and trigger updates by replacing the root value. Virtualize long lists, and use `defineAsyncComponent` for components outside the initial view. Use `v-once` when content no longer changes, and `v-memo` for large subtrees with a confirmed update bottleneck.

**Source:** Vue Docs
