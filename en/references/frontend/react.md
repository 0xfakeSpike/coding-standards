# React

Read the framework-agnostic rules in [components.md](components.md) first. See [state-management.md](state-management.md) for state layering and [data-fetching.md](data-fetching.md) for requests.

## Rendering rules

### Keep components and Hooks pure [Required]

The same Props, State, and Context must produce the same output. During rendering, do not modify external variables, send requests, or read or write `ref.current`. Data fetching in Server Components is the exception. See [nextjs.md](nextjs.md).

**Why:** React may render the same component several times, so side effects run repeatedly and results are unpredictable.

**Tooling:** `react-hooks/purity`, `react-hooks/globals`, `react-hooks/refs`.

**Source:** React Docs

### Treat Props and State as immutable [Required]

Do not modify them directly, and do not modify values after passing them to a Hook or JSX. For example, write `setItems([...items, newItem])`, not `items.push(newItem)`.

**Why:** When the reference does not change, React may not detect the change.

**Tooling:** `react-hooks/immutability`.

**Source:** React Docs

### Do not define components inside components [Required]

Define component functions at the module top level.

**Why:** Every render creates a new component type, so the subtree is unmounted, rebuilt, and loses its state.

**Tooling:** `react-hooks/static-components`.

**Source:** React Docs

### Reset state with `key` [Recommended]

When switching to another entity must clear all state, write `<Chat key={contact.id} contact={contact} />`. Do not clear it manually with an Effect.

**Source:** React Docs

## Hooks

### Enable `eslint-plugin-react-hooks` and call Hooks only at the top level [Required]

Use the `recommended` preset. Call Hooks only at the top level of components or custom Hooks, not in conditions, loops, or nested functions, and not after an early return. Custom Hooks start with `use`.

**Tooling:** `react-hooks/rules-of-hooks`.

**Source:** React Docs

### Use `useSyncExternalStore` for external data sources [Recommended]

Use it to subscribe to third-party Stores or browser state such as `navigator.onLine`. Do not sync manually into State in an Effect. `getSnapshot` must return a cached immutable value, `subscribe` must stay stable, and server-side rendering must provide `getServerSnapshot`.

**Source:** React Docs

## Effect

### Use `useEffect` only to sync with external systems [Recommended]

External systems are subscriptions, timers, DOM operations, third-party components, and network connections. When there is no external system, follow this table.

| Scenario | Approach |
|---|---|
| Derive data from Props or State | Compute during rendering |
| Logic and requests caused by user actions | Put them in event handlers |
| Reset all state when a Prop changes | Pass `key` |
| Notify the parent of a state change | Update both in the same event |
| One-time initialization at app startup | Put it at the module top level or in the entry point |

**Example:**

Avoid:

```tsx
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);
```

Prefer:

```tsx
const fullName = `${firstName} ${lastName}`;
```

**Why:** Unneeded Effects cause extra renders, and chained Effects are hard to trace.

**Tooling:** `react-hooks/set-state-in-effect`.

**Source:** React Docs

### Fill in dependency arrays truthfully [Required]

Do not bypass lint with ignore comments. When dependencies change too often, restructure the code, for example by moving an object into the Effect. Put logic that really should not trigger a re-sync in `useEffectEvent`, which requires React 19.2 or later.

**Why:** Incomplete dependencies read stale closures.

**Tooling:** `react-hooks/exhaustive-deps`.

**Source:** React Docs

### Clean up work started by Effects [Required]

In the cleanup function, stop subscriptions, timers, connections, and requests or ignore their results, so old responses do not overwrite new results. Strict Mode runs one extra mount and unmount cycle in development precisely to expose missing cleanup. Do not work around it with a ref.

```tsx
useEffect(() => {
  const controller = new AbortController();
  loadUser(id, controller.signal).then(setUser, ignoreAbort);
  return () => controller.abort();
}, [id]);
```

**Why:** Missing cleanup causes leaks and race conditions.

**Source:** React Docs

## Performance

### Do not wrap in `useMemo`, `useCallback`, or `memo` by default [Recommended]

Measure first with the React DevTools Profiler or a production build, then prefer structural changes: pass JSX as `children`, keep state as low as possible, and remove unneeded Effects. Add memoization only if a bottleneck remains.

**Why:** Unneeded memoization only makes code harder to read.

**Source:** React Docs

### Do not rely on memoization for correctness [Required]

Code must still behave correctly after you remove `useMemo` and `memo`. Use State or a Ref when you need a stable identity.

**Why:** React discards the cache in certain cases.

**Source:** React Docs

### Use Transitions for non-urgent updates [Recommended]

Put potentially slow updates, such as switching tabs or routes, in `startTransition` so the UI stays responsive. Do not put a controlled input's value in a Transition.

**Source:** React Docs

## Suspense

### Place boundaries by the intended loading order [Recommended]

Put content that appears together in the same boundary. Nest boundaries for content that can appear in stages. Do not wrap every component. On updates, use `startTransition` or `useDeferredValue` to keep displayed content instead of letting the fallback replace it.

**Source:** React Docs

### Only Suspense-enabled data sources trigger boundaries [Required]

`lazy`, Promises read by `use`, and Suspense-enabled frameworks trigger them. Requests in Effects and event handlers do not, so handle their loading state yourself. Cache Promises passed to `use`. Do not create a new one on each render.

**Source:** React Docs

## Error boundaries

### Isolate local errors with error boundaries [Recommended]

Implement error boundaries with class components or `react-error-boundary`. Place them around meaningful areas such as pages and panels, so a local error does not blank the page, and report in `componentDidCatch`. See [observability.md](observability.md).

**Source:** React Docs

### Handle errors outside error boundaries manually [Required]

Error boundaries do not catch errors in event handlers, async callbacks, or server-side rendering. Catch them yourself in those places and update the UI to tell the user. Next.js `error.tsx` can catch Server Component errors. See [nextjs.md](nextjs.md).

**Source:** React Docs

## React Compiler

### Do not hand-write memoization in new code once enabled [Recommended]

Let the compiler handle memoization. Use `useMemo` and `useCallback` only when you need precise control, for example for values used as Effect dependencies. Keep existing hand-written memoization for now, and test before removing it.

**Why:** Removing hand-written memoization may change the compiled output.

**Source:** React Docs

### Fix diagnostics instead of skipping long term [Recommended]

The compiler skips components with diagnostics and compiles the rest as usual. Fix them gradually following ESLint hints. Use `"use no memo"` only for temporary debugging.

**Tooling:** `eslint-plugin-react-hooks`.

**Source:** React Docs
