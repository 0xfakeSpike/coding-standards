# Authentication

This document covers how to confirm "who you are". For "what you can do" once identity is confirmed, see [Authorization](authorization.md). For login rate limiting and credential stuffing protection, see [Audit and abuse prevention](audit-and-abuse.md). For algorithms and keys, see [Cryptography and secrets](crypto-and-secrets.md). For language-specific APIs, see `languages/<language>/security.md`.

## Passwords

### Store passwords with a dedicated password hashing algorithm [Required]

Prefer Argon2id. Use scrypt when it is unavailable, and use bcrypt only when neither is available. Use PBKDF2 when FIPS compliance is required. Do not use MD5, SHA-1, or other fast hashes, and do not use reversible encryption.

| Algorithm | Minimum parameters |
|---|---|
| Argon2id | m=19456 (19 MiB), t=2, p=1 |
| scrypt | N=2^17, r=8, p=1 |
| bcrypt | Work factor at least 10, input at most 72 bytes |
| PBKDF2-HMAC-SHA256 | 600000 iterations |

**Why:** After a database leak, fast hashes can be cracked at scale.

**Source:** OWASP Password Storage Cheat Sheet, NIST SP 800-63B

### Handle long passwords when using bcrypt [Required]

Do not choose bcrypt for new systems. When you must use it, have an explicit plan for long passwords, such as keyed pre-hashing, and state the encoding. Do not truncate silently.

**Why:** 64 Chinese characters take about 192 bytes in UTF-8, over bcrypt's 72-byte limit.

### Base password policy on length and blocklists, not character mix [Recommended]

The minimum length is 15 when a password alone is used to log in, and 8 when the password is only one factor of MFA. Allow at least 64 characters, and allow pasting and password managers. Do not require mixed character types or force periodic password changes. When a password is set, compare the full password against a blocklist of common and leaked passwords. Force a change when there is evidence the password has leaked.

**Source:** NIST SP 800-63B, OWASP ASVS

### Leave no shortcuts in password flows [Required]

**Applies to:** production service

Validate passwords exactly as received, without truncating or changing case. Require the current password to change a password. Do not offer password hints or security questions. Delete or disable default accounts such as `admin` and `root`. System-generated initial passwords are random and short-lived, and cannot serve as long-term passwords.

**Source:** OWASP ASVS

### Do not reveal whether an account exists on authentication failure [Required]

**Applies to:** high risk

When the user does not exist, the password is wrong, or the account is locked, return the same message and status code, and keep response times close. Run one hash computation even when the user does not exist, and do not return early. Password recovery must meet this too. Whether registration can meet this depends on whether there is an email verification flow. When there is one, it must meet this too.

**Why:** Differences in message, status code, or timing can all be used to enumerate accounts.

**Source:** OWASP Authentication Cheat Sheet, OWASP ASVS

## Multi-factor authentication

### Require MFA on high-privilege accounts [Required]

**Applies to:** high risk

Enforce MFA for administrators and other high-privilege accounts. Regular users can at least enable it themselves. Apply the same authentication strength at all login entry points, including separately provided API login.

**Source:** OWASP Multifactor Authentication Cheat Sheet

### Prefer phishing-resistant factors [Recommended]

In priority order: Passkey or WebAuthn, TOTP, SMS. SMS carries SIM hijacking and interception risks. It is a restricted method and is not used for high-value business. Email is not an out-of-band authentication method.

**Source:** NIST SP 800-63B, OWASP Multifactor Authentication Cheat Sheet

### Make one-time codes short-lived, single-use, and attempt-limited [Required]

Generate them with a CSPRNG and invalidate them immediately after successful verification. Out-of-band codes are valid for at most 10 minutes, and TOTP for at most 30 seconds. Bind a code to the request that initiated it, and cap verification attempts. Rate-limit push-based MFA.

**Source:** OWASP ASVS, OWASP Multifactor Authentication Cheat Sheet

### Reauthenticate before changing authentication factors [Required]

**Applies to:** high risk

Before changing an email, phone number, MFA settings, or recovery method, fully authenticate once with an already bound factor. Do not rely on the current session alone. Afterward, notify the user through an out-of-band channel. Recovery codes can be used only once.

**Why:** If a hijacked session can replace factors directly, the account owner gets locked out.

**Source:** OWASP Multifactor Authentication Cheat Sheet, OWASP ASVS

## Sessions

### Generate session IDs on the server with a CSPRNG [Required]

Prefer the framework's session implementation. When generating them yourself, use at least 128 bits of entropy, and keep user information out of the ID. When the threat model includes a read-only leak of the session store, store only token hashes in it.

**Source:** OWASP Session Management Cheat Sheet, OWASP ASVS

### Rotate sessions when authentication state changes [Required]

On login, reauthentication, and privilege elevation, create a new session and invalidate the old one. After a password change or reset, invalidate other sessions. When an account is disabled or deleted, terminate all its sessions.

**Why:** Not rotating allows session fixation attacks, and an attacker may still hold an old session.

**Source:** OWASP Session Management Cheat Sheet, OWASP ASVS

### Set the full security attributes on cookies [Required]

Session cookies must have `HttpOnly`, `Secure`, and an appropriate `SameSite`. Prefer the `__Host-` prefix. Cookie-based authentication also needs CSRF protection. See [Web risks](web-risks.md).

```http
Set-Cookie: __Host-sid=<value>; Path=/; Secure; HttpOnly; SameSite=Lax
```

**Source:** OWASP Session Management Cheat Sheet, OWASP ASVS

### Enforce timeouts and logout on the server [Required]

Set both an idle timeout and an absolute timeout, decided by the server. On logout, destroy the server-side session. Deleting the cookie alone is not enough. Set the specific durations by business risk. For systems held to NIST AAL2, the idle timeout is at most 1 hour and the absolute timeout at most 24 hours. For systems held to AAL3, they are at most 15 minutes and 12 hours respectively.

**Source:** NIST SP 800-63B, OWASP Session Management Cheat Sheet, OWASP ASVS

## JWT

### Decide whether you need JWT first [Recommended]

Prefer server-side sessions for user sessions. Using JWT as a session still requires building your own revocation. When you do use it, keep Access Tokens short-lived and pair them with revocable, rotating Refresh Tokens. When immediate invalidation is needed, build a denylist from `iss` plus `jti`.

**Source:** OWASP JWT Cheat Sheet

### Fix the algorithm and key source on the server [Required]

The verifier configures an algorithm allowlist and does not accept `alg: none`. Use each key for only one algorithm. Do not trust `jku`, `x5u`, or `jwk` in the token header, and take keys only from preconfigured sources. `kid` is external input, so validate it before use. Generate HMAC keys randomly, no shorter than the hash output.

**Why:** Algorithm confusion and tokens that carry their own keys both let forged tokens pass verification.

**Source:** RFC 8725, OWASP JWT Cheat Sheet, OWASP ASVS

### Validate the signature, claims, and token purpose [Required]

Verify the signature first, then read the contents. Validate `exp`, `nbf`, `iss`, and `aud`, and confirm the key belongs to that `iss`. When one issuer has several token types, their validation rules must be mutually exclusive. Mark new types with `typ`, for example `at+jwt`.

**Why:** Otherwise a reset token or ID Token may be accepted as an access token.

**Source:** RFC 8725, OWASP JWT Cheat Sheet, OWASP ASVS

### Keep sensitive information out of the payload [Required]

The payload of a signed JWT is only encoded. Anyone holding the token can read it.

**Source:** OWASP JWT Cheat Sheet

## OAuth 2.1 and OIDC

### Use only the authorization code flow with PKCE [Required]

Do not use the implicit flow or the password flow. Use the `S256` PKCE method, and generate a new value for each transaction. An authorization code can be used only once and is valid for at most 10 minutes. Compare redirect URIs with an exact string match. The only exception is the port number of a local loopback address. Open redirects must not exist.

**Why:** The implicit flow leaks tokens, and the password flow exposes user credentials to the client.

**Source:** RFC 9700, OAuth 2.1 Draft, OWASP ASVS

### Keep tokens out of URLs [Required]

Pass Access Tokens in the `Authorization` header, not in query parameters.

Avoid:

```http
GET /orders?access_token=eyJhbGciOi... HTTP/1.1
```

**Source:** OAuth 2.1 Draft

### Restrict token audience and permissions [Required]

Resource servers validate `aud` on every request and reject mismatches. Clients request only the scopes they need. Refresh Tokens for public clients must be sender-constrained or rotated, and all Refresh Tokens have an absolute expiry. Browser applications prefer a BFF, with tokens held only by the backend.

**Source:** RFC 9700, OWASP ASVS

### Clients fully validate responses and identity [Required]

Confirm the authorization response belongs to a transaction started by the current user agent. Validate the ID Token's signature, `iss`, and `exp`, that `aud` equals your own `client_id`, and that `nonce` matches the request. When integrating with multiple authorization servers, use `iss` to prevent mix-up attacks. Identify users by `iss` plus `sub`, not by email, which can be reassigned.

**Source:** RFC 9700, OWASP ASVS

## Reset flow

### Do not leak information in reset requests [Required]

**Applies to:** high risk

Return the same message whether or not the account exists, and keep timing consistent. Rate-limit per account to prevent email bombing. Do not change account state until a valid token is submitted.

**Source:** OWASP Forgot Password Cheat Sheet

### Make reset tokens random, single-use, and short-lived [Required]

Generate them with a CSPRNG, bind them to a specific user, store only the hash, and invalidate them immediately after use. Build reset links from a fixed domain in configuration, not from the request's `Host` header. Set `Referrer-Policy: no-referrer` on the reset page.

**Why:** When links depend on the `Host` header, an attacker can make a link sent to a victim point to the attacker's domain.

**Source:** OWASP Forgot Password Cheat Sheet

### Finish up after a reset [Required]

**Applies to:** production service

Notify the user that the password changed, without including the password in the notification. Do not log the user in automatically. A reset cannot bypass enabled MFA. Administrators can only initiate a reset. They cannot choose the user's new password.

**Source:** OWASP Forgot Password Cheat Sheet, OWASP ASVS
