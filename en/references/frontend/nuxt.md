# Nuxt

Read [vue.md](vue.md) first. See the [backend layer](../backend/index.md) for general rules on server endpoints. For data fetching, this document takes precedence over [data-fetching.md](data-fetching.md).

## Directory conventions

### Put code in the conventional directories [Recommended]

The `app/` directory structure below requires Nuxt 4 or later.

| Directory | Contents |
|---|---|
| `app/pages/` | File-based routes |
| `app/components/`, `app/composables/`, `app/utils/` | Components and client-side logic, auto-imported |
| `app/layouts/` | Page layouts |
| `app/middleware/` | Code that runs before route navigation |
| `app/plugins/` | Code that runs when the Nuxt app is created |
| `server/` | Server routes and logic |
| `shared/` | Code usable by both the app and the server |
| `public/` | Static files that skip the build |

When organizing by business feature, create per-feature subdirectories under these directories, or split into Nuxt Layers.

**Source:** Nuxt Docs

### Do not import between app code and server code [Required]

Put types and pure functions needed on both sides in `shared/`.

**Why:** The two sides run in different environments, and mixing them pulls server code into the client.

**Source:** Nuxt Docs

### Call Composables synchronously and only in a Nuxt context [Required]

Call them only in `<script setup>`, plugins, and route middleware, not at the top level of a file. In plain functions, do not call them after an `await`.

**Why:** Otherwise you get an error that the Nuxt instance is unavailable.

**Source:** Nuxt Docs

### Set the rendering mode per route with `routeRules` [Recommended]

See [architecture.md](architecture.md) for how to choose.

```ts
export default defineNuxtConfig({
  routeRules: {
    '/pricing': { prerender: true },
    '/blog': { swr: 3600 },
    '/dashboard': { ssr: false },
  },
});
```

**Source:** Nuxt Docs

## Data fetching

### Use `useFetch` or `useAsyncData` for initial page data [Required]

Use `useFetch` for ordinary endpoints, and `useAsyncData` when combining several requests or using a third-party query layer. Write `await useFetch('/api/orders')`, not a direct `await $fetch('/api/orders')` in `setup`. Use `$fetch` only for requests triggered by user actions. When you need a shared baseURL, authorization, and error conversion, create an instance with `$fetch.create` and use it with `createUseFetch` or `useAsyncData`, in place of a shared request client.

**Why:** `$fetch` requests once on the server and again on the client, and can also cause hydration problems. These two functions carry the server result to the client.

**Source:** Nuxt Docs

### Keep `useAsyncData` keys unique and options consistent per key [Required]

Calls with the same key share `data`, `error`, and `status`. Options such as `deep`, `transform`, `pick`, and `default` must match. Otherwise use different keys.

**Source:** Nuxt Docs

### Handle the pending state with `lazy` and `server: false` [Required]

With these two, data may not exist yet at render time. Show loading UI based on `status`.

**Source:** Nuxt Docs

### Shrink the payload with `pick` or `transform` [Recommended]

They reduce the data sent to the client, but not the amount of data fetched by the initial request.

**Source:** Nuxt Docs

### Carry user identity on server-initiated requests [Required]

`useFetch` with a relative URL forwards request headers and cookies automatically. With `$fetch`, pass them manually through `useRequestHeaders(['cookie'])`, or switch to `useRequestFetch`.

**Why:** Otherwise the endpoint cannot see the login state during server-side rendering.

**Source:** Nuxt Docs

## Server routes

### Express path and method in the file name [Recommended]

Files under `server/api/` map to `/api`. Other routes go in `server/routes/`. Add a method suffix to the file name, for example `orders.get.ts`. Use `event.$fetch` for calls between server routes: global `$fetch` does not forward inbound request headers and context.

**Source:** Nuxt Docs

### Read inputs with validating functions [Required]

Use `getValidatedQuery`, `getValidatedRouterParams`, and `readValidatedBody` with schema validation. Passing a schema directly returns 400 on validation failure, but [rest.md](../backend/api/rest.md) and the frontend's field error mapping both expect 422. So pass `safeParse` and return 422 yourself.

```ts
export default defineEventHandler(async (event) => {
  const result = await readValidatedBody(event, orderSchema.safeParse);
  if (!result.success) {
    throw createError({
      status: 422,
      statusText: 'Validation Failed',
      data: { errors: toFieldErrors(result.error) },
    });
  }
  return createOrder(result.data);
});
```

**Source:** Nuxt Docs

### Return error statuses with `createError` [Required]

All uncaught errors return 500. To return another status, write `throw createError({ status: 404, statusText: 'Order Not Found' })`. Status code meanings follow [rest.md](../backend/api/rest.md).

**Source:** Nuxt Docs

### Do not return responses from server middleware [Required]

`server/middleware/` runs before every request. It only inspects or extends `event.context`, or throws an error.

**Source:** Nuxt Docs

## Runtime config

### Declare config in `runtimeConfig` [Required]

Top-level keys are available only on the server. Keys under `public` are sent to the client. Do not put secrets in `public`, and do not pass them to `useState` or templates.

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '',
    public: { apiBase: '' },
  },
});
```

**Source:** Nuxt Docs

### Name environment variables by the `NUXT_` rules [Required]

Use uppercase, start with `NUXT_`, and separate levels with underscores. For example, `NUXT_PUBLIC_API_BASE` overrides `runtimeConfig.public.apiBase`. Variables that do not follow the rules have no effect at runtime. The built server does not read `.env` files, so the runtime environment must provide the variables.

**Source:** Nuxt Docs

### Keep config values serializable [Required]

Do not put functions, `Set`, or `Map` in config. Put that logic in plugins or middleware. Quote secrets that are purely numeric or JSON-like, or they get converted to other types. In server routes use `useRuntimeConfig(event)`, so that runtime environment variable overrides take effect.

**Source:** Nuxt Docs

## State and hydration

### Keep cross-component SSR state out of module-level variables [Required]

Use `useState` or Pinia. Do not export a `ref` at the top level of a module. For example, `export const useUser = () => useState<User | undefined>('user')`. `useState` values must be JSON-serializable.

**Why:** On the server, module-level state is shared by all requests and leaks data.

**Source:** Nuxt Docs

### Eliminate hydration mismatches in SSR-friendly ways [Required]

Use `useCookie` for small persistent values needed in the first SSR render. Restore other browser storage after mount, per [state-management.md](state-management.md). Put content that can only render in the browser in `ClientOnly`. Initialize third-party libraries that manipulate the DOM in `onMounted`.

**Source:** Nuxt Docs

### Integrate Pinia through `@pinia/nuxt` [Recommended]

When session data lives in a Store, write `await callOnce('user', () => store.fetchUser())` in the page to call the fetching Action and avoid duplicate requests. Use `useFetch` for other initial page data. See [state-management.md](state-management.md) for Store boundaries.

**Source:** Pinia Docs
