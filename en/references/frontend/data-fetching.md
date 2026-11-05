# Frontend data fetching

This document covers the browser-side request client, caching, race conditions, retries, optimistic updates, upload and download, and error classification. For Next.js server-side data fetching, see [nextjs.md](nextjs.md). For Nuxt data fetching, see [nuxt.md](nuxt.md). Both take precedence over this document. For async basics, see [concurrency.md](../languages/typescript/concurrency.md). For error handling principles, see [error-handling.md](../general/error-handling.md).

## Request client

### Use one shared request client [Recommended]

Handle baseURL, authorization headers, timeouts, HTTP error detection, and error conversion in one place. Components do not call `fetch` or `axios` directly. Validate responses by the rules in [types.md](../languages/typescript/types.md). 204, 202, and some 201 responses have no body, so declare the schema for those endpoints as empty. Read `Location`, `ETag`, and `Retry-After` from the returned `headers`. Projects that wrap responses in `{ code, message, data }` unwrap them here.

**Why:** `fetch` does not reject on 404 or 500. Checking `response.ok` separately in each place inevitably misses some.

**Example:**

```ts
async function request<T>(
  path: string,
  init: RequestOptions,
  schema: z.ZodType<T>,
): Promise<{ data: T; headers: Headers }> {
  const timeout = AbortSignal.timeout(DEFAULT_TIMEOUT_MS);
  const signal = init.signal ? AbortSignal.any([init.signal, timeout]) : timeout;
  const response = await fetch(`${API_BASE}${path}`, { ...init, signal });
  if (!response.ok) throw await toApiError(response);
  const text = await response.text();
  const data = schema.parse(text === '' ? undefined : JSON.parse(text));
  return { data, headers: response.headers };
}
```

**Source:** MDN

### Organize API functions by business module and declare types [Recommended]

Put API functions in their business module, accept `signal`, and declare the return type, such as `getOrder(id, signal)`. When OpenAPI docs exist, generate types automatically to avoid type drift between frontend and backend.

### Set timeouts and distinguish timeout from cancellation [Required]

`fetch` has no timeout option. Use `AbortSignal.timeout()` or client configuration. A timeout's error name is `TimeoutError`, and the user must be told. An intentional cancellation is `AbortError`, which is not a failure and shows no message.

**Source:** MDN

## Caching and invalidation

### Include every variable that affects the result in the cache key [Required]

Put every variable the request function uses into the key. A TanStack Query key is a serializable array. Element order matters, and object property order does not.

**Why:** When a variable is not in the key, different parameters hit the same cache entry and show wrong data.

**Example:**

```ts
useQuery({
  queryKey: ['orders', { status }],
  queryFn: ({ signal }) => listOrders(status, signal),
});
```

**Source:** TanStack Query Docs, SWR Docs

### Set cache time by how often data changes [Recommended]

TanStack Query treats cached data as stale by default and refreshes in the background on mount, window focus, and network reconnect. Unused cache entries are collected after 5 minutes. SWR also revalidates on focus and reconnect by default. Set `staleTime` for slow-changing data, and use `useSWRImmutable` for data that does not change.

**Source:** TanStack Query Docs, SWR Docs

### Invalidate related queries after writes [Required]

After a successful write, invalidate related queries with `invalidateQueries`, or update the cache with the API response. Use the same prefix for keys of the same resource, so `['orders']` matches all queries under that prefix. Otherwise the page keeps showing pre-write data.

**Source:** TanStack Query Docs

## Race conditions

### Handle request race conditions [Required]

In scenarios such as search boxes and tab switching, an old request may return after a new one and overwrite the new result. Manage this with a request library, and pass `signal` to the request. TanStack Query truly cancels only when `signal` is used. When writing by hand, cancel the old request with `AbortController`, creating a new one per request.

**Example:**

```ts
useEffect(() => {
  const controller = new AbortController();
  search(keyword, controller.signal).then(setResults, ignoreAbort);
  return () => controller.abort();
}, [keyword]);
```

**Source:** MDN, TanStack Query Docs

## Retries

### Auto-retry only idempotent requests [Recommended]

POST and PATCH are not guaranteed idempotent, so do not retry them automatically, except critical writes with `Idempotency-Key`. TanStack Query mutations do not retry by default, so do not turn that on globally. Queries retry automatically by default. Retrying 4xx other than 429 is pointless, so exclude them with a `retry` function. When a query gets 429 or 503 with `Retry-After`, retry after that wait time. For cross-origin requests, the backend must expose this response header. See [web-risks.md](../backend/security/web-risks.md).

**Source:** MDN, TanStack Query Docs

### Reuse the idempotency key when retrying critical writes [Required]

**Applies to:** high risk

Critical writes generate `Idempotency-Key` at the start of one logical submission, and the shared request layer sends it. Manual retries after timeouts and network errors reuse the same key. Change the key only after the user changes the content. While a request with the same key is still running, the backend returns 409. In that case, tell the user to try again later. For backend rules, see [rest.md](../backend/api/rest.md).

**Why:** After a timeout, you cannot tell whether the request succeeded. With a new key on retry, the backend treats it as a new request, which creates duplicate orders.

## Optimistic updates

### Use optimistic updates only for easily rolled-back operations [Recommended]

Likes, checkboxes, and sorting are a good fit. Do not use them for operations the backend must confirm, such as payment and placing orders. When the data appears in only one place, render a temporary item from the mutation's `variables` and `isPending`, without changing the cache. Change the cache only when multiple places need to see it.

**Source:** TanStack Query Docs

### Complete cancel, snapshot, rollback, and invalidation when changing the cache [Required]

All four steps are required. SWR uses `optimisticData` with `rollbackOnError`, which is on by default.

**Why:** Without canceling in-flight requests, old responses overwrite the optimistic value. Without rollback, the UI stays in a wrong state after failure. Without invalidation, the cache diverges from the server.

**Example:**

```ts
useMutation({
  mutationFn: updateTodo,
  onMutate: async (next) => {
    await queryClient.cancelQueries({ queryKey: todoKeys.detail(next.id) });
    const previous = queryClient.getQueryData<Todo>(todoKeys.detail(next.id));
    queryClient.setQueryData(todoKeys.detail(next.id), next);
    return { previous };
  },
  onError: (_error, next, context) => {
    queryClient.setQueryData(todoKeys.detail(next.id), context?.previous);
  },
  onSettled: (_data, _error, next) =>
    queryClient.invalidateQueries({ queryKey: todoKeys.detail(next.id) }),
});
```

**Source:** TanStack Query Docs, SWR Docs

## Upload and download

### Do not set `Content-Type` by hand for `FormData` uploads [Required]

The request client's default JSON headers must not apply to `FormData` either. If it is set by hand, the browser cannot write the boundary, and the server cannot parse the body. Frontend type and size checks are only for user experience. For backend rules, see [file-handling.md](../backend/security/file-handling.md).

**Source:** MDN

### Use `XMLHttpRequest` when you need upload progress [Recommended]

Listen to the `progress` event on `xhr.upload`, and connect `AbortSignal` to `xhr.abort()`. For cross-origin requests, registering a listener triggers a preflight request.

**Source:** MDN, TanStack Query Docs

### Release object URLs after Blob downloads [Recommended]

After the download triggers, call `URL.revokeObjectURL()` to release URLs created with `URL.createObjectURL()`. Otherwise memory leaks. For large files, prefer letting the browser access the download URL directly instead of reading the file into memory.

**Source:** MDN

## Error classification

### Handle errors by type [Recommended]

The request client converts errors to one shared type. Common handling is centralized, and business errors go to the caller. For route-level display, see [routing.md](routing.md). For the backend error format, see [rest.md](../backend/api/rest.md).

| Error | Handling |
|---|---|
| 400 | Generic message, and report to error monitoring |
| 401 | Refresh credentials or redirect to login in one place, and keep the return URL. Concurrent requests share one refresh |
| 403 | Show a no-permission message |
| 404 | Show that the resource does not exist |
| 409 | Distinguish by error identifier. For business conflicts, show the reason. For an idempotent request still running, tell the user to try again later |
| 412 | Tell the user the data has changed, and reload |
| 422 | Show field or business errors. For field mapping, see [forms.md](forms.md) |
| 429 | Queries follow the retry rules above. Writes tell the user to try again later |
| 5xx, network errors | Generic message with a retry option. Critical writes reuse the idempotency key |

### Do not show technical errors to users [Required]

Stack traces and SQL errors stay out of the UI. Error messages must tell users what they can do next. Report technical details to error monitoring such as Sentry. Upload Source Maps but do not deploy them publicly. See [observability.md](observability.md).

## Field conventions

For instants and local dates, see [i18n.md](i18n.md). For money and 64-bit integer arithmetic, see [idioms.md](../languages/typescript/idioms.md). For API field conventions, see [rest.md](../backend/api/rest.md).

For environment variables, see [build-and-compat.md](build-and-compat.md).
