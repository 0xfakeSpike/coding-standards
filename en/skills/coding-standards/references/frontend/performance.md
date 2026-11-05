# Frontend performance

This document covers browser-side performance targets and optimizations. See [Performance](../general/performance.md) for general measurement methods, [Responsive design and assets](responsive-and-assets.md) for images and fonts, [Build and compatibility](build-and-compat.md) for bundling configuration, and [Observability](observability.md) for production collection.

## Core Web Vitals

### Use Core Web Vitals as experience targets [Recommended]

**Applies to:** production service

| Metric | Measures | Good | Needs improvement | Poor |
|---|---|---|---|---|
| LCP | Loading | <= 2.5 s | 2.5 to 4.0 s | > 4.0 s |
| INP | Interaction responsiveness | <= 200 ms | 200 to 500 ms | > 500 ms |
| CLS | Visual stability | <= 0.1 | 0.1 to 0.25 | > 0.25 |

Measure at the 75th percentile and assess mobile and desktop separately. All three must reach "Good" to pass.

**Source:** web.dev Core Web Vitals, web.dev LCP, web.dev INP, web.dev CLS

### Treat real user data as authoritative [Recommended]

**Applies to:** production service

Judge whether targets are met from real user data in production. See [Observability](observability.md) for how to collect it. Use Lighthouse, the browser Performance panel, and React and Vue DevTools to locate bottlenecks. Do not optimize by feel. Lighthouse cannot measure INP, and TBT serves only as a substitute reference.

**Why:** Lab devices and networks are fixed, so they miss the real user distribution and real interactions.

**Source:** web.dev Core Web Vitals

### Measure performance changes the same way before and after [Required]

Use the same environment, the same data, and the same steps before and after a change, and record the actual numbers. See [Performance](../general/performance.md) for how.

**Source:** Team

## Loading optimization

### Make LCP resources discoverable early [Recommended]

Put the LCP image directly in the initial HTML. When it is referenced only in CSS or JavaScript, load it early with `preload`. Set `fetchpriority="high"` on it, and use this attribute on only one or two images.

Avoid:

```html
<div class="hero" data-bg="/hero.avif"></div>
```

Prefer:

```html
<img src="/hero.avif" width="1200" height="600" fetchpriority="high" alt="Main product image">
```

**Why:** The browser finds the image only after scripts or styles run, and that delay counts directly toward LCP.

**Source:** web.dev: Optimize LCP

### Do not lazy-load above-the-fold images [Recommended]

Use `loading="lazy"` only for images outside the initial viewport. Keep the default loading behavior for images that may appear above the fold.

**Source:** web.dev: Optimize LCP, web.dev: Lazy Loading Images

### Load large dependencies on demand [Recommended]

Dynamically load large dependencies such as charts and editors when they are used. Import only the modules you use from third-party libraries. See [routing.md](routing.md) for route splitting.

**Source:** web.dev: Code Splitting

### Reduce render-blocking resources [Recommended]

Do not put synchronous scripts in `<head>`. Use `defer` instead. Shrink render-blocking stylesheets. Load embedded content outside the initial view when it scrolls nearby. Use `preconnect` only for the most critical third-party origins, and `dns-prefetch` for secondary ones.

**Source:** web.dev: Optimize LCP, web.dev: Loading Third-Party Scripts

## Rendering optimization

### Reserve space for late content [Recommended]

See [Responsive design and assets](responsive-and-assets.md) for image dimensions. Hold space for ads, embedded content, and dynamic banners with `min-height` or `aspect-ratio`. Keep the placeholder when no content comes back, because collapsing it also causes a shift. Animate with `transform`, not layout-triggering properties such as `top` and `left`.

**Source:** web.dev: Optimize CLS

### Use virtual scrolling or pagination for long lists [Recommended]

When a list is large (usually a thousand items or more), render only the visible part, or paginate.

**Why:** Rendering many nodes at once causes visible jank.

**Source:** web.dev: Virtualizing Long Lists

### Do not interleave layout reads and writes in one task [Recommended]

Batch reads first, then batch writes.

**Why:** Reading a layout value right after changing styles forces synchronous layout.

**Source:** web.dev: Optimize INP

### Control DOM size [Recommended]

Reduce nesting, and add elements only on interaction. Large offscreen content can defer rendering with `content-visibility`.

**Source:** web.dev: Optimize INP

## Long tasks

### Split long tasks [Recommended]

A task over 50 ms is a long task. Split work into batches and yield to the main thread after about 50 ms of accumulated work. Do not yield after every small task. Prefer `scheduler.yield()`, and fall back to `setTimeout` when it is not supported.

```ts
function yieldToMain(): Promise<void> {
  if (globalThis.scheduler?.yield) {
    return scheduler.yield();
  }
  return new Promise((resolve) => setTimeout(resolve, 0));
}

async function processAll(items: Item[]) {
  let deadline = performance.now() + 50;
  for (const item of items) {
    processItem(item);
    if (performance.now() >= deadline) {
      await yieldToMain();
      deadline = performance.now() + 50;
    }
  }
}
```

**Why:** While the main thread is occupied, user input gets no response and INP worsens.

**Source:** web.dev: Optimize Long Tasks

### Update the UI first in event callbacks [Recommended]

Callbacks do only the work needed to update the UI. Move work such as saving and analytics to the next task.

**Source:** web.dev: Optimize INP

### Move compute-heavy work to a Web Worker [Recommended]

Move work that does not touch the DOM, such as parsing large files and complex calculations, to a Worker to avoid blocking the main thread.

## Caching

### Cache hashed assets long term and revalidate HTML every time [Recommended]

**Applies to:** production service

Give static assets with a content hash in the file name a one-year cache. The HTML entry file name does not change, so revalidate it before every use. See [Build and compatibility](build-and-compat.md) for release configuration.

| Resource | Response header |
|---|---|
| Hashed static assets | `Cache-Control: max-age=31536000, immutable` |
| Static HTML entry | `Cache-Control: no-cache` |
| SSR pages with personal data | `Cache-Control: no-store`, see [Web risks](../backend/security/web-risks.md) |

**Why:** Once HTML is cached long term, users cannot get the new version.

**Source:** web.dev: HTTP Cache

### Keep pages eligible for the back/forward cache [Recommended]

**Applies to:** production service

Do not use the `unload` event. Use `pagehide` instead. Register `beforeunload` only when there is unsaved content, and remove it after saving. Use `no-store` only for pages with personal or other sensitive data. Refresh stale data in `pageshow` based on `event.persisted`.

**Why:** The back/forward cache makes back and forward navigation nearly instant, and avoids repeated layout shifts.

**Source:** web.dev: Back/Forward Cache

## Performance budgets

### Set budgets by page type and enforce them in CI [Recommended]

**Applies to:** production service, team project

Limit both resource sizes and user-facing timing metrics, with separate budgets for different page types. For example, limit JavaScript size on product pages and image size on search pages. Fail the build when a budget is exceeded. Optimize or remove resources. Do not simply loosen the budget.

```js
module.exports = {
  ci: {
    collect: { numberOfRuns: 5 },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
      },
    },
  },
};
```

**Tooling:** Lighthouse CI

**Source:** web.dev: Performance Budgets, web.dev Lighthouse CI
