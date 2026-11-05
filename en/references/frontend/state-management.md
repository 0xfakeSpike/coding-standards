# Frontend state management

This document covers where state lives, how to design Stores, and which state to persist or put in the URL. See [data-fetching.md](data-fetching.md) for request caching, [forms.md](forms.md) for form state, and [components.md](components.md) for UI state inside components.

## State layering

### Choose where state lives by its nature [Recommended]

Do not put all data in a global Store. Keep local state in the nearest component, and lift it only when several non-adjacent components really need to share it.

| State type | Examples | Where it lives |
|---|---|---|
| Server data | Lists, details | TanStack Query, SWR, Pinia Colada, Next.js Server Component, Nuxt `useFetch`, etc. |
| Shareable page state | Filters, pagination, tabs | URL query params |
| Local UI state | Dialog open state, input values | Inside the component |
| Cross-component client state | Theme, current user | Zustand, Pinia, Context |
| Form state | Fields, validation | React Hook Form, VeeValidate, etc. |

**Source:** Redux Style Guide

### Keep one source of truth per piece of data [Required]

Server data stays in the request cache and URL state stays in the URL. Other places only read it and do not copy it. Global session data such as the logged-in user may be held by a Store. The Store is then its source of truth, and the request cache does not keep a copy.

**Why:** Copies require manual syncing, and two copies eventually diverge.

**Example:**

Avoid:

```ts
useEffect(() => {
  if (data) useOrderStore.getState().setOrder(data);
}, [data]);
```

Prefer:

```ts
const { data: order } = useQuery(orderQuery(id));
```

### Use a data fetching library for server data [Recommended]

Writing caching, deduplication, retries, and invalidation yourself is slow and error-prone. Leave them to a fetching library, not the Store.

**Source:** Redux Style Guide, TanStack Query Docs

## Store design

### Split Stores by business domain [Recommended]

Divide by data type or business domain, such as `auth` and `cart`. Do not divide by page, and do not build a giant Store that holds everything. In Pinia, use one Store per business domain. Zustand and Redux officially recommend a single Store, split internally into slices by business domain.

**Why:** The same data is often read by several pages, so dividing by page causes duplication and unclear ownership.

**Source:** Redux Style Guide, Zustand Docs, Pinia Docs

### Centralize state updates in Store actions [Recommended]

Keep actions together with the state, and have components only call actions. Zustand updates with `set`. Redux does not modify state directly, and actions are named after what happened.

**Why:** When updates scatter across components, you cannot find what changed the state, and subscribers may miss notifications.

**Source:** Redux Style Guide, Zustand Docs

### Use serializable plain data in state fields [Recommended]

Do not put Promises, class instances, or DOM nodes in state fields, or debugging tools and persistence stop working. Redux state and actions must follow this, and must not hold `Map` or `Set` either. Action functions in Zustand and Pinia are not state fields.

**Source:** Redux Style Guide

### Select only the fields you need when subscribing [Recommended]

Components do not subscribe to the whole Store. When a Zustand selector returns a new array or object, wrap it in `useShallow`. See [vue.md](vue.md) for Pinia destructuring.

**Example:**

```ts
const { items, total } = useCartStore(
  useShallow((state) => ({ items: state.items, total: state.total })),
);
```

**Source:** Zustand Docs, Pinia Docs, Redux Style Guide

See [vue.md](vue.md) for Pinia constraints.

## Choosing a solution

### Use one solution per layer in a project [Recommended]

| Scenario | Choice |
|---|---|
| Vue client state | Pinia |
| Small to medium React client state | Zustand |
| React project already using Redux | Redux Toolkit, with RTK Query for server data |
| Low-frequency dependency injection, such as theme | Context, `provide` |

**Source:** Redux Style Guide, Pinia Docs

## Derived state

### Do not store values you can derive [Recommended]

Store only raw data. Compute totals, filter results, and select-all status at read time. Memoize expensive computations: in Redux use `createSelector`, with selectors named `selectXxx`; in Vue use `computed` or a Pinia getter; see [react.md](react.md) for React components.

**Why:** Stored derived values must be synced at every write point, and missing one causes a bug.

**Source:** Redux Style Guide

## Persistence

### Persist only fields still needed after a refresh [Recommended]

Select them with `partialize` in Zustand's `persist`. Keep sensitive information out of browser storage. See [security.md](security.md).

**Source:** Zustand Docs

### Centralize storage reads and writes, and tolerate failures [Recommended]

Keep `localStorage` access and writes in one module, inside `try/catch`. Validate the format after reading, and treat failure as no data.

**Why:** Access throws `SecurityError` when storage is disabled, writes throw `QuotaExceededError` when space is full, and existing data may have the wrong format.

**Example:**

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

**Source:** MDN, Zod Docs

### Prefix storage keys with the project name [Recommended]

For example `myapp:theme`, so apps on the same domain do not overwrite each other.

**Source:** Zustand Docs

### Handle storage restore timing and cross-tab sync [Recommended]

The server cannot read browser storage. Render the first screen with initial values and restore after mount. In Zustand, use `skipHydration` with `rehydrate()`. For cross-tab sync, listen to the `storage` event, which does not fire on the page that made the change.

**Source:** Zustand Docs, MDN

## URL state

### Keep shareable page state in the URL [Recommended]

Put filters, page numbers, sorting, and tabs in query params, and do not keep another copy in the Store. When an API uses cursor pagination, do not put the cursor in the URL, and clear the cursor when filters change.

**Why:** In memory, the state is lost on refresh, link sharing, and back navigation.

### Validate URL params as external data [Required]

Users can modify URLs freely. Parse and validate on read, and fall back to defaults for invalid values. See [types.md](../languages/typescript/types.md) for the rules.

**Example:**

```ts
const listParamsSchema = z.object({
  page: z.coerce.number().int().positive().catch(1),
  status: z.enum(['open', 'closed']).catch('open'),
});
```

**Source:** Zod Docs

### Choose push or replace by back-button expectations [Recommended]

Use push for changes users expect to go back through, such as pagination and tabs. Use replace for params that keep changing during input, for example `setSearchParams({ keyword }, { replace: true })`. React Router's `setSearchParams` does not queue, so update multiple params in one call.

**Source:** MDN, React Router Docs

### Keep sensitive information out of URLs [Required]

Keep passwords, tokens, and personal information out of paths and query params. They stay in history and logs, and spread when shared.

**Source:** web.dev: Learn Forms
