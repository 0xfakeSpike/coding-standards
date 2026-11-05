# Frontend routing

This document covers client-side route design, guards, lazy loading, scroll restoration, and error pages. See [state-management.md](state-management.md) for URL query params, and [nextjs.md](nextjs.md) and [nuxt.md](nuxt.md) for Next.js and Nuxt file-based routing.

## Route design

### Define routes centrally and express layouts with nested routes [Recommended]

Keep route configuration in one place. In Vue Router, put permissions, titles, and similar information on `meta`, and add types by declaring `RouteMeta`. Put shared shells in layout routes, and render child pages through `<Outlet />` or `<RouterView />`.

**Source:** Vue Router Docs, React Router Docs

### Respond to route param changes [Required]

When switching from `/users/1` to `/users/2`, the component instance is reused, logic that runs only on mount does not run again, and the page still shows old data. Put the param in the cache key, or watch the param and cancel the old request.

**Example:**

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

**Source:** Vue Router Docs

## Guards

### Centralize login and permission redirects in a global route guard [Recommended]

Routes declare what they require. The global guard checks and redirects in one place, and page components do not each handle it. Vue Router uses `beforeEach` with `meta`. React Router uses `middleware` on a protected parent route, and declarative routes use a protected layout route. See [nextjs.md](nextjs.md) and [nuxt.md](nuxt.md) for the Next.js and Nuxt approaches.

**Why:** With checks scattered across pages, new pages easily miss them, and redirect rules cannot be changed in one place.

**Example:**

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

**Source:** Vue Router Docs, React Router Docs

### Guards affect experience only; the backend is the authority on permissions [Required]

Guards, hidden menus, and hidden buttons decide only what users see. The backend validates permissions on every endpoint. When the frontend receives 401 or 403, handle it per [data-fetching.md](data-fetching.md).

**Why:** Client logic is easy to bypass. Users can modify the code or call endpoints directly.

**Source:** OWASP Authorization Cheat Sheet

### Do not create loops or hangs in guards [Required]

Redirects must exclude the target page. The login page must not trigger a redirect to the login page, or you get an infinite redirect. Vue Router guards control flow through return values: `false` cancels, a route location redirects, and returning nothing proceeds. Do not declare a `next` parameter. Once it is declared, the router waits for it to be called, and navigation may hang.

**Source:** Vue Router Docs

### Accept only same-site paths for post-login redirects [Required]

The `redirect` param comes from the URL, and users can craft anything. Parse it with a URL parser and compare origins. Do not check with string prefixes: `startsWith('/')` does not stop URLs that begin with a double slash.

**Why:** Without validation, users land on an attacker's site after login.

**Example:**

```ts
function safeRedirectPath(raw: string | null): string {
  const origin = window.location.origin;
  if (raw === null || !URL.canParse(raw, origin)) return '/';
  const url = new URL(raw, origin);
  return url.origin === origin ? `${url.pathname}${url.search}${url.hash}` : '/';
}
```

**Source:** OWASP Unvalidated Redirects Cheat Sheet

## Lazy loading

### Lazy-load route components with dynamic `import()` [Recommended]

Use dynamic `import()` for route components by default. Bundlers split the code automatically. Static imports are fine when there are few pages. In Vue Router, pass a function that returns a Promise to `component`. Do not wrap route components in `defineAsyncComponent`. In React Router, use the route's `lazy` field. Declarative routes use `React.lazy`, declared at the module top level and wrapped in `Suspense`.

**Why:** Static imports put all pages in the initial bundle.

**Example:**

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

**Tooling:** Use build output analysis to confirm that page code is not in the entry bundle. See [build-and-compat.md](build-and-compat.md).

**Source:** Vue Router Docs, React Router Docs, React Docs

### Handle page code load failures [Recommended]

**Applies to:** production service

Dynamic imports fail when the network drops or files are removed after a release. In React, the error goes to the nearest error boundary. In Vue Router, handle it with `router.onError`. The UI must offer a way to refresh. Otherwise clicking a link does nothing, or a blank page appears.

**Source:** React Docs, Vue Router Docs

## Scroll restoration

### Restore position on back and go to the top on new pages [Recommended]

Route changes in a single-page app do not reload the page. If this is not handled, a new page stays at the previous page's scroll position. Configure `scrollBehavior` in Vue Router. In React Router Framework and Data modes, render one `<ScrollRestoration />`.

**Example:**

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

**Source:** Vue Router Docs, React Router Docs

### Do not reset scroll on in-page changes, and let one owner manage scrolling [Recommended]

Tabs and filters that change only query params do not go back to the top. Use `preventScrollReset` in React Router. Once the router library takes over, do not also hand-write `window.scrollTo` or modify `history.scrollRestoration`.

**Source:** React Router Docs, Vue Router Docs, MDN

## 404 and permission pages

### Configure a fallback 404 route [Recommended]

Use `/:pathMatch(.*)*` in Vue Router and `path: '*'` in React Router. Placing it inside a layout route keeps the shell. In History mode, the server must fall back to the entry page for unknown paths.

**Source:** Vue Router Docs, React Router Docs

### Distinguish missing route, missing resource, and no permission [Recommended]

Render error pages in place at the current URL. Do not redirect to `/404`.

| Case | Handling |
|---|---|
| URL matches no route | 404 page |
| Route exists, API returns 404 | Show resource not found in the current route |
| API returns 403 | Permission page, no redirect to login |
| API returns 401 | Redirect to login and keep the return URL |

**Why:** A redirect loses the URL, so users cannot refresh to retry or report the failing address.

### Configure a root error boundary [Recommended]

In React Router, export at least one root `ErrorBoundary` that handles response errors, `Error` instances, and other thrown values separately. Do not show stack traces in production. Error boundaries are for unexpected errors. Do not throw for normal flow control. The exception is a missing resource: use `throw data(message, { status: 404 })` in a loader. See [forms.md](forms.md) for form validation errors.

**Source:** React Router Docs

### Update the page title after navigation [Recommended]

Read the route's title information in a global after hook, such as Vue Router's `afterEach`. See [accessibility.md](accessibility.md) for focus management after route changes.

**Source:** Vue Router Docs
