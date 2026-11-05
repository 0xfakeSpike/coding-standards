# Frontend components

The rules in this document do not depend on a specific framework. For framework details, see [react.md](react.md) and [vue.md](vue.md). For directories and module boundaries, see [architecture.md](architecture.md).

## Component categories

### Separate components by responsibility [Recommended]

| Category | Responsibility | Can fetch data | Location |
|---|---|---|---|
| Base component | UI with no business meaning, such as buttons and inputs | No | `components/` or the design system |
| Layout component | Page frame and region arrangement | No | `components/` |
| Business component | Displays and operates on a business object | As needed | `features/` |
| Page component | Assembles business components and connects to routing | Yes | Route entry |

### Give each component one responsibility [Recommended]

Consider splitting or switching to composition when you see these signals:

- Describing the component requires the word "and".
- Props mix several unrelated groups of configuration.
- Multiple boolean Props switch between different forms of the component.
- Changing one use case often affects another.
- One component handles requests, business decisions, and display at once.
- A section of the UI has independent state and can be named on its own.

**Why:** Components with mixed responsibilities are hard to reuse and test, and changing them is risky.

### Keep generic components free of business dependencies, and business logic outside components [Recommended]

Generic components receive data through Props and do not call business APIs directly. Put business logic in a Hook, Composable, or plain function, and let the component focus on rendering, so the logic is easy to reuse and test.

## Naming

### Use PascalCase for component names [Recommended]

For example, `UserCard`. Callback props start with `on`, and internal handlers start with `handle`, such as `onSubmit` and `handleSubmit`. For additional Vue requirements, see [vue.md](vue.md).

## Share one set for similar components

### Share one set for similar components [Required]

When two places have the same kind of reader, share one set of components, and express differences through Props or slots. Before adding a component, search for an existing implementation. If it can be extended, do not write another.

**Example:**

Avoid:

```text
features/contract/components/ContractReader.tsx
features/report/components/ReportReader.tsx
```

Prefer:

```tsx
<DocumentReader source={contract.fileUrl} toolbar={<ContractActions />} />
<DocumentReader source={report.fileUrl} toolbar={<ReportActions />} />
```

**Why:** Multiple implementations eventually diverge, and a fix in one place easily misses the other.

**Exceptions:** Components that only look similar, but differ in business meaning and reasons to change.

**Source:** Team

## Props design

### Pass only the data a component needs [Recommended]

Props contain only fields the component actually uses. Do not pass the whole business object down.

**Why:** Smaller dependencies are easier to reuse and less likely to carry sensitive fields where they should not go.

### Use one enum Prop for mutually exclusive forms [Recommended]

Write `<Button variant="danger" />`, not `<Button primary danger ghost />`. For how to write the type, see [types.md](../languages/typescript/types.md).

**Why:** Multiple booleans can combine into states that do not exist.

### Do not mutate passed-in data in child components [Required]

Data flows down, and changes notify upward through callbacks or events. Do not change internal fields of object and array Props either.

**Why:** Only one-way data flow lets you trace where state changes.

**Source:** React Docs, Vue Style Guide

### Do not copy Props into state [Required]

Use the Prop directly, or compute from it during render.

**Why:** After copying, only the initial value takes effect, and later updates from the parent are ignored.

**Exceptions:** When you truly want only the initial value, show the intent with a name such as `initialValue` or `defaultValue`.

**Source:** React Docs, Vue Docs

## Composition patterns

### Use composition to solve prop drilling [Recommended]

When an intermediate layer only forwards props, prefer `children` or `slot` so that the data owner renders the consumer directly. Use Context or `provide`/`inject` only when composition is not enough.

**Example:**

Avoid:

```tsx
<Layout user={user} posts={posts} />
```

Prefer:

```tsx
<Layout header={<UserMenu user={user} />}>
  <PostList posts={posts} />
</Layout>
```

**Source:** React Docs

### Use subcomponents instead of grouped config Props [Recommended]

When the content and structure of an area change often, provide subcomponents or slots such as `CardHeader` and `CardFooter`. Do not keep adding config Props such as `showIcon` and `footerText`.

## Controlled and uncontrolled

### Lift shared state to the common parent [Recommended]

When multiple components need the same state, lift it to the nearest common parent, then pass it down. For requirements on the authoritative source, see [state-management.md](state-management.md).

**Source:** React Docs

### Use controlled components when coordination is needed [Recommended]

Use the controlled approach when the parent needs to read, validate, or link a value. Use the uncontrolled approach when state concerns only the component itself. When supporting both, name them separately: controlled uses `value` with `onChange`, uncontrolled uses `defaultValue`, and Vue components use `v-model`. For the trade-offs for form controls, see [forms.md](forms.md).

**Source:** React Docs

## UI states

### Handle all UI states [Required]

Components that show async data must handle loading, error, and success states. Lists also handle empty data. When the framework or request library already provides Suspense, a `loading` file, or `status`, use it directly. When writing state by hand, use one discriminated union field, not multiple booleans. The error UI says what happened and offers a retry or a way back.

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
    throw new Error(`Unhandled state ${JSON.stringify(unhandled)}`);
  }
}
```

**Why:** Implementing only the success state is the most common omission. Users see a blank screen or an empty table.

**Source:** React Docs

## Lists and derived data

### Use stable business IDs for list `key` [Required]

Do not use array indexes or random numbers. Assign IDs to locally created data at creation, not during render.

**Why:** On insertion, deletion, and sorting, unstable `key` values misplace state or make components rebuild repeatedly.

**Exceptions:** Static lists that never have items added, removed, or sorted may use indexes.

**Source:** React Docs, Vue Style Guide

Do not store computable values as state. See [state-management.md](state-management.md).

## Component documentation

### Document shared components in Storybook or project docs [Recommended]

**Applies to:** team project

Base and cross-feature components must have docs covering purpose, Props, events, slots, and usage limits. When the team already uses Storybook, generate doc pages with the `autodocs` tag. Otherwise write them in project docs.

**Why:** Consumers discover existing components by checking docs first, which avoids duplicate implementations.

**Source:** Storybook Docs

### Write one Story per state [Recommended]

Keep Stories next to component files. Each named export corresponds to one state, covering the main variants and the loading, empty, and error states. Story files stay out of the production bundle.

**Source:** Storybook Docs
