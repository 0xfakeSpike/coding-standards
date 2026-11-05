# TypeScript / JavaScript security

For general backend security, see [backend security rules](../../backend/index.md). This document covers only language and runtime risks.

## Language-specific risks

### Do not execute dynamic string code [Required]

Do not use `eval`, `new Function`, or the string form of `setTimeout`. The `vm` module is not a security boundary and cannot run untrusted code.

**Tooling:** ESLint `no-eval`, `no-new-func`, `@typescript-eslint/no-implied-eval`.

**Source:** Google TypeScript Style Guide, Node.js API Docs, MDN

### Prevent prototype pollution [Required]

Attackers rewrite prototypes through `__proto__`, `constructor`, and `prototype`, affecting every object in the process, which can bypass permission checks or cause denial of service. Keep untrusted data out of recursive merges and `obj[key] = value`.

**Example:**

Avoid:

```ts
function setByPath(target: Record<string, any>, path: string[], value: unknown): void {
  const lastKey = path.at(-1)!;
  const parent = path.slice(0, -1).reduce((node, key) => (node[key] ??= {}), target);
  parent[lastKey] = value;
}
```

Prefer:

```ts
const preferencesSchema = z.strictObject({ theme: z.enum(['light', 'dark']) });
const preferences = preferencesSchema.parse(JSON.parse(body));
```

**Source:** Node.js Security Best Practices, MDN

### Use safe containers for dynamic keys [Required]

When keys come from outside, use `Map`, `Set`, or `Object.create(null)`. When you must write to a plain object, first check the key against an allowlist, or exclude `__proto__`, `constructor`, and `prototype`. High-risk services can add `--disable-proto=delete`. It does not cover the `constructor.prototype` path and does not replace validation.

**Source:** MDN, Node.js Security Best Practices

### Read only own properties for permission fields [Recommended]

A polluted prototype makes a read such as `user.isAdmin` return the attacker's value. Use `Object.hasOwn` before checking. Iterate with `Object.keys`, not `for...in`.

**Source:** MDN

### No untrusted patterns in regular expressions [Required]

Avoid nested quantifiers `(a+)*`, overlapping alternatives `(a|a)*`, and backreferences. Do not build regular expressions from user input. For simple matching use `includes` or `indexOf`. ReDoS blocks the event loop and affects all requests. `JSON.parse` also runs synchronously, so set a body size limit at the framework layer first.

**Source:** Node.js Learn Guides

### Confine file paths to the root directory [Required]

Do not concatenate user input directly into paths. For the requirements, see [file-handling.md](../../backend/security/file-handling.md). After `realpath` resolves symbolic links, confirm the result is under the allowed root directory before accessing it. `uploadRoot` is a resolved absolute path.

**Example:**

```ts
const filePath = await realpath(path.resolve(uploadRoot, fileName));
if (!filePath.startsWith(uploadRoot + path.sep)) {
  throw new InvalidPathError(fileName);
}
const content = await readFile(filePath);
```

## Random numbers

### Use cryptographic randomness in security contexts [Required]

Use `crypto.randomUUID()`, `crypto.randomBytes()`, `crypto.randomInt()`, or `crypto.getRandomValues()` for tokens, session IDs, verification codes, and salts. `Math.random()` is predictable, so use it only where there are no security requirements.

**Example:**

Avoid:

```ts
const resetToken = Math.random().toString(36).slice(2);
```

Prefer:

```ts
const resetToken = randomUUID();
```

**Source:** MDN, Node.js API Docs

### Compare secrets with a constant-time function [Required]

`===` comparison time varies with content and can be used to guess the value. Use `crypto.timingSafeEqual`. The two Buffers must have the same length.

**Source:** Node.js Security Best Practices, Node.js API Docs

### Use async APIs for password hashing [Required]

For algorithm order and parameters, see [authentication.md](../../backend/security/authentication.md). In Node.js, do not use the sync versions: `scryptSync` and `pbkdf2Sync` block the event loop on the request path.

**Source:** Node.js Security Best Practices, Node.js Learn Guides

## Serialization

### Parse untrusted data with JSON only [Required]

`JSON.parse` produces only data. Do not use `v8.deserialize`, YAML custom types, or libraries that can restore functions on untrusted input.

### Validate parsed results [Required]

`JSON.parse` returns `any`, and the `__proto__` it creates is an own property, which can reach the prototype during a later `Object.assign` or spread merge. Validate with a schema right after parsing and reject extra fields, for example with Zod's `z.strictObject`.

**Source:** Node.js Security Best Practices, MDN

### Do not serialize whole objects to logs and responses [Required]

Objects may contain passwords and tokens. Output responses with explicit fields or a serialization schema, and use a redaction config for logs. Fastify's validation and serialization compile schemas with `new Function()`, so schemas must not come from user input.

**Source:** Fastify Docs

## Command execution

### Pass argument arrays to `execFile` or `spawn` [Required]

`exec` and `shell: true` hand the string to the shell, so user input can inject commands. Values starting with `-` may be treated as options. Separate them with `--` when the program supports it. Set a timeout, and do not use `execSync` or `spawnSync` on the request path.

**Example:**

Avoid:

```ts
await execAsync(`gzip ${fileName}`);
```

Prefer:

```ts
await execFileAsync('gzip', ['--', fileName], { signal, timeout: gzipTimeoutMs });
```

**Source:** Node.js API Docs, Node.js Learn Guides

## Static scanning

### Do not enable the debug port in production [Required]

Do not enable `--inspect` in production.

**Source:** Node.js Security Best Practices

### Include dependency vulnerabilities in CI [Recommended]

**Applies to:** team project

CI runs `npm audit --audit-level=high` or `pnpm audit`, and verifies signatures with `npm audit signatures`.

**Source:** npm Docs, Node.js Security Best Practices

### Guard against supply chain attacks when installing dependencies [Required]

**Applies to:** production service

Check package names to avoid typosquatting, and pin all transitive dependencies with a lock file. For install script restrictions, see [project.md](project.md). pnpm can delay adoption of new versions with `minimumReleaseAge`.

**Source:** Node.js Security Best Practices, pnpm Docs
