# Frontend security

This document covers only what the browser side must do. See [Web risks](../backend/security/web-risks.md) for server-side CSRF protection and security response headers, [Authentication](../backend/security/authentication.md) for cookie attributes, and [TypeScript security](../languages/typescript/security.md) for `eval` and prototype pollution. Dependency vulnerability scanning is also in that file.

## XSS

### Prefer framework escaping and text writes [Required]

Output untrusted data through the framework's default templates, or write it to `textContent`. Do not write it to `innerHTML`.

Avoid:

```ts
element.innerHTML = comment;
```

Prefer:

```ts
element.textContent = comment;
```

**Source:** OWASP XSS Prevention Cheat Sheet, OWASP DOM-Based XSS Prevention Cheat Sheet

### Do not write unsanitized content into HTML [Required]

`dangerouslySetInnerHTML`, `v-html`, and `innerHTML` are all XSS entry points. When you must render rich text, sanitize it with DOMPurify first.

```tsx
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
```

Sanitization is the last step before writing to the DOM. Modifying content after sanitizing voids the sanitization. Upgrade the sanitizer library promptly. When running on the server, use the latest jsdom, not happy-dom.

**Tooling:** `no-danger` from `eslint-plugin-react`, `no-v-html` from `eslint-plugin-vue`

**Source:** OWASP XSS Prevention Cheat Sheet, DOMPurify Docs

### Validate the protocol of user-provided URLs [Required]

Parse URLs before writing them to `href` or `src`, and allow only allowlisted protocols. Framework blocking of `javascript:` does not replace validation.

```ts
const ALLOWED_PROTOCOLS = new Set(['http:', 'https:']);

function toSafeHref(input: string): string | undefined {
  try {
    const url = new URL(input, window.location.origin);
    return ALLOWED_PROTOCOLS.has(url.protocol) ? url.href : undefined;
  } catch {
    return undefined;
  }
}
```

**Source:** OWASP XSS Prevention Cheat Sheet

### Do not write untrusted data into event attributes or style source [Required]

Do not use `setAttribute` to write attributes such as `onclick`. Bind events with `addEventListener`. Do not concatenate untrusted data into CSS selectors or style blocks.

**Why:** Event attribute values execute as code, and encoding does not stop them.

**Source:** OWASP DOM-Based XSS Prevention Cheat Sheet, OWASP XSS Prevention Cheat Sheet

### Tighten write points with Trusted Types [Recommended]

Set `require-trusted-types-for 'script'` in CSP so write points accept only policy-generated values.

**Source:** MDN

## CSP

### Send CSP with all HTML responses [Required]

**Applies to:** production service

The policy must include at least `frame-ancestors`, `object-src`, and `base-uri`. See [Web risks](../backend/security/web-risks.md) for response header requirements.

### Use a strict CSP for scripts [Recommended]

Prefer nonce-based or hash-based policies. Do not maintain domain allowlists. Regenerate the nonce on every response, which requires dynamically generated HTML. Use hashes for purely static deployments. Code must not rely on `unsafe-inline` or `unsafe-eval`: replace inline event attributes and `javascript:` links with script bindings, and do not execute code from strings. CSP is defense in depth and does not replace the XSS rules above.

```http
Content-Security-Policy: script-src 'nonce-{RANDOM}' 'strict-dynamic'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
```

**Source:** web.dev: Strict CSP, OWASP Content Security Policy Cheat Sheet, MDN

### Ship with Report-Only first [Recommended]

Observe violations of a new policy through the `Content-Security-Policy-Report-Only` response header, and enforce it after fixing them. `<meta>` does not support Report-Only or `frame-ancestors`.

**Source:** web.dev: Strict CSP, OWASP Content Security Policy Cheat Sheet

## CSRF

### Handle cookie-authenticated write requests with the backend's chosen CSRF scheme [Required]

Handle it in the shared request layer, not per endpoint. When the backend uses tokens, write the token to a custom request header, and keep it out of the URL. When the backend uses Fetch Metadata plus Origin validation, the frontend needs no token. See [Web risks](../backend/security/web-risks.md) for server-side protection.

```ts
await fetch('/api/orders', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'X-CSRF-Token': csrfToken },
  body: JSON.stringify(order),
});
```

**Source:** OWASP CSRF Prevention Cheat Sheet

### Use `credentials: 'include'` only for cross-origin requests [Recommended]

By default, `fetch` sends cookies only on same-origin requests. Switch to `include` only for endpoints that really must send credentials cross-origin. Do not set it globally.

## Credentials and trust boundaries

### Prefer `HttpOnly` cookies for login state [Recommended]

Use cookies with `HttpOnly`, `Secure`, and `SameSite`. You must then also defend against CSRF.

**Why:** Tokens in `localStorage` can be read directly during XSS.

**Source:** OWASP HTML5 Security Cheat Sheet, MDN

### Do not keep sensitive information in Web Storage [Required]

Do not store session identifiers, tokens, or personal information in `localStorage`, `sessionStorage`, or IndexedDB.

**Exceptions:** When Bearer tokens are required, keep them only in memory.

**Source:** OWASP HTML5 Security Cheat Sheet

### Keep secrets out of the frontend; the backend is the authority on permissions [Required]

Bundles are visible to everyone. Route calls that need secrets through the backend. See [Build and compatibility](build-and-compat.md) for environment variables. Hiding a button on the frontend is only an experience improvement, not a security measure.

## Third-party scripts

### Load third-party scripts asynchronously and on demand [Recommended]

Load them with `async` or `defer`, and include them only on pages that need them.

**Source:** web.dev: Loading Third-Party Scripts, OWASP Third-Party JavaScript Management Cheat Sheet

### Configure SRI for CDN scripts [Recommended]

**Applies to:** production service

Set `integrity` and `crossorigin` on scripts loaded from a CDN, and use pinned-version URLs. Self-host critical scripts whose content changes and so cannot use SRI.

```html
<script src="https://cdn.example.com/lib@1.2.3/lib.min.js" integrity="sha384-{HASH}" crossorigin="anonymous"></script>
```

**Why:** If CDN content is tampered with, the script runs with the page's privileges.

**Source:** MDN, OWASP Third-Party JavaScript Management Cheat Sheet

### Limit the data third parties can see [Recommended]

Feed tag managers through your own data layer. Do not let them read the DOM freely. Put high-risk third-party code in a `sandbox` iframe on a different domain, and communicate through `postMessage`.

**Source:** OWASP Third-Party JavaScript Management Cheat Sheet

## Clickjacking

### Confirm pages cannot be embedded by other sites [Required]

**Applies to:** production service

Set both CSP `frame-ancestors` and `X-Frame-Options` with response headers. See [Web risks](../backend/security/web-risks.md) for values. Setting them in `<meta>` has no effect, and JavaScript frame-busting scripts can be bypassed, so neither counts as protection. Pages that must be embeddable list only trusted origins.

**Source:** OWASP Clickjacking Defense Cheat Sheet, MDN

## Source maps

### Do not deploy source maps publicly [Recommended]

**Applies to:** production service

For production builds, do not set `build.sourcemap` to `'inline'`, and do not deploy directly with `true`. Upload source maps only to the error monitoring platform. After upload, delete them from the deploy directory, or block access to `.map` files on the server. Vite's `build.sourcemap: 'hidden'` only removes the reference comment from files, and `.map` files are still generated. See [Observability](observability.md) for configuration.

**Why:** Public source maps leak source code.

**Source:** Sentry JavaScript Docs, Vite Docs

## Cross-window communication

### Validate origins with `postMessage` [Required]

Specify the target origin when sending. Do not use `*`. When receiving, validate `event.origin` and the message content. Do not register a listener when you do not need to receive external messages.

```ts
window.addEventListener('message', (event) => {
  if (event.origin !== TRUSTED_ORIGIN) return;
  handleMessage(parseMessage(event.data));
});
```

**Source:** MDN

### Cut `opener` when opening untrusted sites [Recommended]

Add `rel="noopener noreferrer"` when `target="_blank"` opens an untrusted site, and pass `noopener` to `window.open`.

**Source:** OWASP HTML5 Security Cheat Sheet, MDN
