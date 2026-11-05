# Web risks

This document covers common risks in browser-server interaction. For cookie attributes, see [Authentication](authentication.md). For TLS configuration, see [Cryptography and secrets](crypto-and-secrets.md). For callback and webhook signing and replay protection, see [Webhook](../api/webhooks.md).

## CORS

### Use an explicit origin allowlist [Required]

Set `Access-Control-Allow-Origin` to a fixed value, or return `Origin` after a full string comparison against the allowlist. Do not reflect arbitrary `Origin` values, do not use prefix or suffix matching, and do not allow `null`.

Avoid:

```http
Access-Control-Allow-Origin: <Origin from the request, returned as is>
Access-Control-Allow-Credentials: true
```

Prefer:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

**Why:** With a reflected origin plus credentials, any website can read responses as the user.

**Source:** OWASP ASVS, MDN

### Do not use wildcards with credentials [Required]

When allowing credentials, use explicit values for origins, methods, and request headers. Use `*` only for public resources with no sensitive information. When returning the origin dynamically, add `Vary: Origin` to avoid cache mix-ups.

**Source:** MDN, OWASP ASVS, OWASP HTTP Headers Cheat Sheet

### CORS is not access control [Required]

CORS only restricts browsers from reading responses. Simple requests still reach the server and execute. The server must do authentication, authorization, and CSRF protection on its own.

**Source:** OWASP ASVS

### Expose response headers the frontend reads [Recommended]

In cross-origin requests, scripts can read only CORS-safelisted response headers. List headers such as `Retry-After`, `ETag`, `Location`, and the request identifier in `Access-Control-Expose-Headers`.

## CSRF

### Cookie-based authentication must prevent CSRF [Required]

Prefer the framework's built-in protection. When there is none, choose from the table below and apply it to all state-changing requests.

| Approach | Use when |
|---|---|
| Synchronizer token | There are server-side sessions |
| Signed double-submit cookie, token bound to the session | Stateless services |
| Fetch Metadata with Origin validation | Modern browsers only, HTTPS site-wide |
| Custom request header | APIs that do not use forms |

How Fetch Metadata works: for unsafe methods, decide by `Sec-Fetch-Site`. Allow `same-origin` and reject `cross-site`. Reject `same-site` by default, unless you fully control all same-site subdomains. For `none`, allow only top-level navigation the user initiated directly. When a request lacks the header, fall back to `Origin` or `Referer` validation, and reject outright on sensitive endpoints.

**Why:** Browsers attach cookies automatically, so a third-party page can make a user send a request unknowingly.

**Exceptions:** Endpoints that pass tokens only in the `Authorization` header and do not use cookie authentication are not affected by CSRF.

**Source:** OWASP CSRF Prevention Cheat Sheet

### `GET` does not change state [Required]

Use `POST`, `PUT`, `PATCH`, or `DELETE` for state-changing operations.

**Why:** `SameSite=Lax` and most CSRF protections let safe methods through.

**Source:** OWASP CSRF Prevention Cheat Sheet, OWASP ASVS

### `SameSite` is only defense in depth [Recommended]

`SameSite` decides by registrable domain, so same-site subdomains are not restricted, and browser defaults differ. Unless you fully control all same-site subdomains, still pair it with tokens. Do not use unsigned naive double-submit cookies. XSS can bypass all CSRF protection, and neither can replace the other.

**Source:** OWASP CSRF Prevention Cheat Sheet

## SSRF

### Use an allowlist when targets are known [Required]

When the server accesses only known services, let the user choose an identifier, and let the server map it to an address. Do not accept URLs. When you must accept one, validate against an allowlist of scheme, host, port, and path, and compare hosts strictly.

**Source:** OWASP SSRF Prevention Cheat Sheet, OWASP ASVS

### Block internal addresses when targets are not fixed [Required]

For cases such as webhooks that can reach arbitrary external addresses, allow only `http` and `https`, resolve all A and AAAA records, and reject the following addresses:

| Type | Examples |
|---|---|
| Loopback | `127.0.0.0/8`, `::1` |
| Private ranges | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `fc00::/7` |
| Link-local | `169.254.0.0/16`, `fe80::/10` |
| Cloud metadata | `169.254.169.254`, `metadata.google.internal` |

Use a mature library to parse IPs, to prevent bypasses through encodings such as hexadecimal and octal.

**Source:** OWASP SSRF Prevention Cheat Sheet

### Defend against redirects and DNS rebinding [Required]

Turn off automatic redirects in the HTTP client. When redirects are truly needed, revalidate at each hop. After validation, connect directly to the validated IP, while keeping the original hostname for `Host`, SNI, and certificate validation. No unchecked DNS resolution may happen between validation and connection.

**Why:** When only the domain name is validated, an attacker can make it resolve to an internal address after validation.

**Source:** OWASP SSRF Prevention Cheat Sheet

### Restrict outbound traffic at the network layer [Recommended]

Put components that make external requests in a restricted subnet, and use a firewall or outbound proxy to open only necessary destinations. On cloud hosts, enable a metadata service that requires tokens, such as AWS IMDSv2. Do not return fetched responses to users as is.

**Source:** OWASP SSRF Prevention Cheat Sheet

## Open redirects

### Do not use user input as a redirect target [Required]

Prefer having the user pass a short identifier that the server maps to an address. For on-site redirects, accept only local paths, using the framework's validation method.

Avoid:

```http
GET /login?next=https://evil.example/phish HTTP/1.1
```

Prefer:

```http
GET /login?next=orders HTTP/1.1
```

**Why:** Open redirects can be used for phishing, and can also steal OAuth authorization codes and tokens.

**Source:** OWASP Unvalidated Redirects Cheat Sheet, RFC 9700

### Parse external targets and compare against an allowlist [Required]

Use a maintained URL parser, and compare the scheme, canonicalized host, and port. Reject addresses with user info, such as `https://good.example@evil.example`. Do not use prefix or suffix matching. Redirect using the validated URL, and allow only `http` and `https`. For redirects off-site, prefer an interstitial page where the user confirms.

**Source:** OWASP Unvalidated Redirects Cheat Sheet, OWASP ASVS

## Security response headers

### Set the baseline response headers [Required]

**Applies to:** production service

| Header | Recommended value | Purpose |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Enforce HTTPS, for at least one year |
| `X-Content-Type-Options` | `nosniff` | Forbid content type sniffing |
| `Content-Security-Policy` | HTML responses must include `frame-ancestors`, `object-src`, and `base-uri`. For a strict `script-src`, see [Frontend security](../../frontend/security.md) | Prevent clickjacking, restrict plugins and the base URL, tighten script sources |
| `X-Frame-Options` | `DENY`, for other values see the next rule | Fallback for browsers that do not support `frame-ancestors` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer information leakage |
| `Content-Type` | Matches the content, with `charset` | Prevent incorrect rendering |
| `Cache-Control` | `no-store` on responses with personal or other sensitive data | Prevent caching |

`no-cache` does not prevent storage, so it cannot replace `no-store` on these responses. HSTS `includeSubDomains` and `preload` are hard to undo quickly if misconfigured. Enable them only after confirming all subdomains support HTTPS.

**Tooling:** OWASP ZAP

**Source:** OWASP HTTP Headers Cheat Sheet, OWASP ASVS

### Use `frame-ancestors` and `X-Frame-Options` against clickjacking [Required]

**Applies to:** production service

`frame-ancestors` is the main protection, and `X-Frame-Options` is the fallback for old browsers. Set both through response headers. When both are present, browsers that support `frame-ancestors` ignore `X-Frame-Options`.

| Embedding need | `frame-ancestors` | `X-Frame-Options` |
|---|---|---|
| No embedding (default) | `'none'` | `DENY` |
| Same-origin embedding only | `'self'` | `SAMEORIGIN` |
| Trusted cross-origin embedding | List the trusted origins, such as `https://partner.example` | Do not send, modern browsers do not support `ALLOW-FROM` |

**Source:** MDN

### Set isolation headers on document responses [Recommended]

Set `Cross-Origin-Opener-Policy: same-origin` on HTML responses. Use `Permissions-Policy` to turn off unused browser capabilities, such as `geolocation=(), camera=()`.

**Source:** OWASP HTTP Headers Cheat Sheet, OWASP ASVS

### Clean up unnecessary response headers [Recommended]

Leave `X-XSS-Protection` unset or set it to `0`, and rely on CSP. Do not use `Expect-CT`. Remove `X-Powered-By`, and keep version information out of `Server`.

**Source:** OWASP HTTP Headers Cheat Sheet
