# Next.js

This document targets the App Router. Read [react.md](react.md) first. Caching rules assume `cacheComponents` is enabled. For server-side data fetching and error handling, this document takes precedence over the client-rendering rules in [data-fetching.md](data-fetching.md) and react.md.

## App Router

### Express route states with file conventions [Recommended]

Use `layout` for shared layouts, `page` for public pages, `loading` for loading UI, `error` and `global-error` for uncaught exceptions, `not-found` for 404s, and `route` for endpoints. Keep route-internal files in private directories such as `_components`. Use `(group)` to group routes without affecting the URL.

**Source:** Next.js Docs

### `await` `params` and `searchParams`, and treat them as user input [Required]

In Next.js 15 and later they are Promises. They are also external input that anyone can modify. Validate them before use, and do not use them to decide permissions.

```tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  return <Order id={id} />;
}
```

**Source:** Next.js Docs

### Separate expected errors from uncaught exceptions [Recommended]

Handle expected errors as return values and display them with `useActionState`. Call `notFound()` when a resource does not exist. Throw program defects directly and let `error.tsx` catch them. That file must be a Client Component. See [forms.md](forms.md) for form details.

**Source:** Next.js Docs

### Narrow `proxy.ts` with `matcher` [Recommended]

`proxy.ts` requires Next.js 16 or later. Without `matcher`, it intercepts all requests, including static assets. It cannot be the only authorization mechanism.

**Source:** Next.js Docs

## Server and Client Components

### Default to Server Components and push `'use client'` to the leaves [Recommended]

Mark only components that need State, events, Effects, or browser APIs. For example, when only the search box in a layout needs interaction, mark only the file containing `Search`, not the whole `Layout`.

**Why:** Everything a marked file imports goes into the client bundle.

**Source:** Next.js Docs

### Pass Server Components into Client Components with `children` [Recommended]

Do not import Server Components inside Client Components. Receive them through `children` or another Prop, for example `<Modal><Cart /></Modal>`. Also wrap Context Providers as Client Components that accept `children`, and place them as deep as possible.

**Source:** Next.js Docs

### Keep Props passed to Client Components serializable and minimal [Required]

Pass only the fields needed for rendering. Do not pass database records or full user objects.

**Example:**

Avoid:

```tsx
<Profile user={userRow} />
```

Prefer:

```tsx
<Profile name={userRow.name} avatarUrl={userRow.avatarUrl} />
```

**Why:** Every field passed to a Client Component is sent to the browser.

**Source:** Next.js Docs

### Import `server-only` in server-only modules [Required]

Write `import 'server-only'` at the top of modules that contain secrets or database access.

**Why:** If client code imports the module by mistake, the build fails immediately.

**Source:** Next.js Docs

## Data fetching and caching

See [data-fetching.md](data-fetching.md) for general request rules.

### Fetch data in Server Components near where it is used [Recommended]

Identical `fetch` requests are deduplicated automatically, so you can request directly in the components that need the data. A `fetch` with a `signal` is not deduplicated. Wrap requests that need a timeout in `React.cache`, as you do ORM and database queries. Start independent requests in parallel with `Promise.all`. Do not `await` them one after another.

**Source:** Next.js Docs

### Put slow data and request-time data in `<Suspense>` [Recommended]

Wrap components that read `cookies()`, `headers()`, `searchParams`, or uncached data in a nearby `<Suspense>`, and output the rest of the content first. Do not read this data directly in a `layout`: a `loading` at the same level does not cover it.

**Source:** Next.js Docs

### Use a data access layer in new projects [Recommended]

Build a `server-only` data access layer. Do permission checks there and return only minimal data objects. Choose one data fetching approach per project.

**Why:** Centralizing keeps permission checks consistent and gives security audits a clear entry point.

**Source:** Next.js Docs

### Do not mutate data during rendering [Required]

Put logout, database writes, and cache invalidation in Server Actions or Route Handlers.

**Source:** Next.js Docs

### Set the lifetime of `use cache` with `cacheLife` [Recommended]

Without it, the implicit `default` profile applies and the call site does not show how long data is cached. Do not cache data that must be fresh on every request.

```tsx
export async function getProducts() {
  'use cache';
  cacheLife('hours');
  cacheTag('products');
  return db.query('SELECT * FROM products');
}
```

**Source:** Next.js Docs

### Do not read request data inside a cache scope [Required]

`cookies()`, `headers()`, and `searchParams` cannot be read inside `use cache`, including in functions it calls. Read them outside and pass them in as arguments. The arguments become part of the cache key.

**Why:** These errors can pass the build and fail only at runtime.

**Source:** Next.js Docs

### Choose the cache update method by scenario [Recommended]

Prefer invalidating by tag, which is more precise than by path. `updateTag` and `refresh` require Next.js 16 or later.

| Method | Use when |
|---|---|
| `updateTag` | Users must see their own changes immediately. Server Actions only |
| `revalidateTag(tag, 'max')` | Brief staleness is acceptable. Refreshes in the background |
| `revalidatePath` | Only one route is affected and a tag is not worth adding |
| `refresh` | Refresh the current route only, without invalidating the cache |

**Source:** Next.js Docs

## Server Actions

### Check identity and permissions inside every Action [Required]

Actions can be called directly with a POST. Page-level checks, `proxy.ts`, and merely rendering the form are not security boundaries. Check the user's permission on the resource as well as login state.

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

**Source:** Next.js Docs

### Validate inputs and control return values [Required]

`FormData` and all other inputs are untrusted. The client passes only IDs and the user's changes. Reread everything else from a trusted source. Return only the fields the UI needs.

**Why:** Return values are serialized and sent to the client, and schema validation does not prove resource ownership.

**Source:** Next.js Docs

### Use Actions only for mutations [Recommended]

Read data in Server Components. The client dispatches Actions one at a time in order, even with `Promise.all`. When you need parallelism, do it inside a single Action.

**Source:** Next.js Docs

### Refresh the cache before `redirect` [Required]

`redirect` interrupts execution by throwing an exception, so code after it does not run.

**Source:** Next.js Docs

## Environment variables

See [build-and-compat.md](build-and-compat.md) for frontend environment variable requirements. Variables without the `NEXT_PUBLIC_` prefix are available only on the server. Read secrets only in the data access layer.

### Treat public variables as build-time constants [Required]

`NEXT_PUBLIC_` variables are inlined at build time, and later changes have no effect. Write the full `process.env.NEXT_PUBLIC_ANALYTICS_ID`. Reading by a dynamic variable name does not get replaced. When deploying the same image to several environments, read runtime variables on the server.

**Source:** Next.js Docs

### Keep `.env` files out of version control [Required]

**Exceptions:** `.env.test` without secrets may be committed. Do not commit `.env*.local`.

**Source:** Next.js Docs
