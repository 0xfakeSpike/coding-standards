# Frontend observability

This document covers browser-side error monitoring, performance collection, and instrumentation. See [Performance](performance.md) for performance targets, [Security](security.md) for source map security requirements, [Privacy](../backend/security/privacy.md) for personal data requirements, and the [backend standards](../backend/index.md) for server logs and tracing.

**Applies to:** production service (all rules in this document)

## Error monitoring

### Integrate an error monitoring platform [Recommended]

In production, integrate a platform such as Sentry. Initialize it as early as possible, and capture synchronous exceptions and unhandled Promise rejections. If you build it yourself, listen to both `error` and `unhandledrejection`: the former does not receive unhandled Promise rejections.

**Source:** Sentry JavaScript Docs, MDN

### Tie errors to release and environment [Recommended]

Set `release` explicitly on every deployment, and set `environment` with consistent names.

```ts
Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  release: import.meta.env.VITE_RELEASE,
  environment: import.meta.env.MODE,
});
```

**Why:** Without a release, you cannot tell which release introduced an error.

**Source:** Sentry JavaScript Docs

### Upload source maps at build time [Recommended]

Upload from CI with a build plugin, for production builds only. The upload must happen before errors occur. Keep the upload token in CI secrets, without a `VITE_` prefix. After upload, delete `.map` files with the plugin's `sourcemaps.filesToDeleteAfterUpload`.

```ts
export default defineConfig({
  build: { sourcemap: 'hidden' },
  plugins: [
    sentryVitePlugin({
      org: 'example-org',
      project: 'example-web',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      sourcemaps: { filesToDeleteAfterUpload: ['dist/assets/*.map'] },
    }),
  ],
});
```

**Source:** Sentry JavaScript Docs

### Filter noise instead of sampling to control volume [Recommended]

Use `ignoreErrors`, `allowUrls`, and `beforeSend` to exclude errors from browser extensions and third-party scripts. When error volume is too high, prefer the platform's rate limiting: the SDK's `sampleRate` also reduces visibility into real problems.

**Source:** Sentry JavaScript Docs

### Report only exceptions that need action [Recommended]

Do not report expected outcomes, such as form validation failures and user-canceled requests, as errors. Treat handled responses such as 401 and 404 as described in [Data fetching](data-fetching.md).

### Remove sensitive data before reporting [Required]

Clean it with `beforeSend` and `beforeBreadcrumb` before it leaves the browser. Report users by internal ID only. Platform-side scrubbing happens after the data is sent and cannot replace this step.

Avoid:

```ts
Sentry.setUser({ email: user.email });
```

Prefer:

```ts
Sentry.setUser({ id: user.id });
```

**Source:** Sentry JavaScript Docs

### Send trace headers only to your own backend [Recommended]

List only your own backend domains in `tracePropagationTargets`, so trace data does not go to third parties. For cross-origin requests, confirm that backend CORS allows these headers. Confirm that the trace headers the SDK sends match the format the backend uses. The backend entry point treats them as untrusted input. See [Distributed tracing](../backend/observability/tracing.md).

**Source:** Sentry JavaScript Docs

## Performance monitoring

### Collect Core Web Vitals from real users [Recommended]

Collect LCP, INP, and CLS with the `web-vitals` library. Register callbacks only once per page. Use `web-vitals/attribution` to find causes.

```ts
function send(metric: Metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    name: metric.name,
    value: metric.value,
    id: metric.id,
    route: currentRouteTemplate(),
  }));
}

onLCP(send);
onINP(send);
onCLS(send);
```

**Why:** Duplicate registration creates extra observers.

**Source:** web-vitals Docs, web.dev Core Web Vitals

### Account for when metrics are reported [Recommended]

CLS and INP report only when the page is hidden, and report again after a restore from the back/forward cache. There is no INP when the user does not interact. Deduplicate by `id` on the server and use the last value.

**Source:** web-vitals Docs

### Group by route template and device [Recommended]

Look at the 75th percentile, with mobile and desktop separate. Use route templates such as `/orders/:id`, not actual URLs.

**Why:** Actual URLs have too high a cardinality and may include user data.

**Source:** web.dev Core Web Vitals

### Track route changes separately in single-page apps [Recommended]

Route changes are not full page navigations, so Core Web Vitals do not reflect them. Record route change duration separately when needed.

**Source:** web-vitals Docs

### Start performance tracing at a low sample rate [Recommended]

Start `tracesSampleRate` low and adjust once you understand the traffic. `tracesSampler` can set different rates per operation.

**Source:** Sentry JavaScript Docs

## Instrumentation rules

### Define events in one place [Recommended]

Define types for event names and properties centrally. Use a fixed naming format, such as `object_action`.

Avoid:

```ts
track('click1', { p: 2 });
```

Prefer:

```ts
track('order_submitted', { itemCount: 2, paymentType: 'card' });
```

**Why:** Events without a shared definition quickly lose their analytical value.

### Report through a single entry point [Recommended]

Business code calls only the project's `track` wrapper, not third-party SDKs directly. Constrain events and properties with types. Put consent checks and field filtering in this layer.

**Why:** Scattered calls cannot be governed consistently, and changing platforms means changing all the code.

### Keep instrumentation from affecting business behavior [Required]

A failed report does not throw and does not block user actions. Do not delay page navigation to wait for reporting. Use `navigator.sendBeacon()` when leaving the page.

**Source:** MDN

### Isolate environments [Recommended]

Events from development and test environments do not enter production data.

## Privacy requirements

### Confirm the basis before collecting [Required]

Load analytics and third-party SDKs that require consent only after the user consents. Stop collecting after consent is withdrawn. Record consent by purpose. Include monitoring and analytics platforms in the data inventory as processors. See [Privacy](../backend/security/privacy.md).

### Do not collect direct identifiers or input content [Required]

Keep names, phone numbers, emails, tokens, and form input values out of event properties, error messages, and breadcrumbs. Strip query parameters from URLs before reporting, or report only the route template. Identify users by internal ID or pseudonym. These still count as personal information, so manage them per [Privacy](../backend/security/privacy.md).

### Keep default masking in session replay [Required]

Keep `maskAllText`, `maskAllInputs`, and `blockAllMedia` on. When there is a real need, open up precisely with `unmask` and `unblock`. Put masking attributes on shared components such as avatars and input fields. CSS background images are outside media blocking, so add `data-sentry-block` manually. Verify again after changing the config or upgrading the SDK.

**Source:** Sentry JavaScript Docs
