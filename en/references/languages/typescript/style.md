# TypeScript / JavaScript style

Applies to Node.js services, libraries, and scripts. For React, Vue, and CSS, see the frontend layer.

## Tool configuration

### tsconfig baseline [Recommended]

New projects use the config below by default. `strict` does not include the other checks here, so enable them one by one. Adjust `module`, `rootDir`, and `types` for the project environment. `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` are costly and can be turned off per project. When they are on, prefer `for...of` and `entries()` for iteration, and do not write index access.

**Why:** `noUncheckedIndexedAccess` exposes out-of-bounds access at compile time. `exactOptionalPropertyTypes` distinguishes "no such property" from "value is `undefined`". `verbatimModuleSyntax` emits import statements as written, so single-file transpilers get consistent results.

**Example:**

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "module": "nodenext",
    "target": "es2023",
    "rootDir": "./src",
    "outDir": "./dist",
    "types": ["node"]
  },
  "include": ["src"]
}
```

**Exceptions:** Projects using a bundler set `module` to `preserve` or `esnext`, and `moduleResolution` to `bundler`.

**Tooling:** `tsc --noEmit`, in CI.

**Source:** TSConfig Reference, TypeScript Handbook

### Use a typed lint preset [Recommended]

**Applies to:** team project

Use typescript-eslint's `recommendedTypeChecked` with ESLint, and upgrade to `strictTypeChecked` when the team can accept it. Rules such as `no-floating-promises`, `no-misused-promises`, and `await-thenable` rely on type information, and presets without types do not enable them.

**Tooling:** `eslint` or Biome, in CI.

**Source:** typescript-eslint Docs

### Do not require doc comments in lint config [Required]

Do not enable rules that require doc comments, such as `jsdoc/require-jsdoc`, except for published libraries. Write a one-line reason after `@ts-expect-error`, which `ban-ts-comment` requires by default.

**Source:** Team

### No `console.log` in business code [Recommended]

Send output that needs to stay through the project's logging module, and enable ESLint `no-console`. Normal output of command-line tools is exempt.

**Why:** Debug output bypasses log levels and redaction, and easily leaves sensitive data in production.

## Formatting

### Leave formatting to the formatter [Recommended]

**Applies to:** team project

Use Prettier or Biome, one per project. Do not enable ESLint formatting rules, and do not discuss formatting in review. Do not rely on automatic semicolon insertion. Use single quotes for strings. Keep the config in the repository and check it in CI.

**Tooling:** `prettier --check .` or `biome check`.

**Source:** typescript-eslint Docs, Google TypeScript Style Guide

### Comments record only what the code cannot say [Required]

Do not write comments that restate the code, including doc comments that only repeat the name and type, except API docs for published libraries and license headers. When code needs comments to be understood, improve the naming and structure. Tool directives such as `@ts-expect-error` and `eslint-disable-next-line` do not count as comments. When the code cannot express a reason, write it in one line next to the code, such as an external system limit, counter-intuitive behavior, or a single-point trade-off like ignoring an error, a type assertion, or suppressing lint.

**Source:** Team

## Naming

### Use case by identifier kind [Recommended]

| Item | Rule | Example |
|---|---|---|
| Variables, functions, methods, properties | `camelCase` | `orderTotal`, `fetchUser` |
| Classes, interfaces, types, enums | `PascalCase` | `OrderService` |
| Module-level constants | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Generic parameters | `T` or a descriptive `PascalCase` name | `TItem` |
| Files | `kebab-case`, or the project's convention | `order-service.ts` |

Use `UPPER_SNAKE_CASE` only for module-level values that do not change during the program's lifetime. Do not prefix interfaces with `I` or private members with an underscore. Use an underscore prefix only for bindings that the syntax requires but you do not use. `number` cannot express units, so write `timeoutMs` and `priceCents`. Start booleans with `is`, `has`, `can`, or `should`, start functions with a verb, and use plurals for collections.

**Tooling:** `@typescript-eslint/naming-convention`.

**Source:** Google TypeScript Style Guide

### Treat abbreviations as words [Recommended]

Write `loadHttpUrl` and `customerId`, not `loadHTTPURL` and `customerID`. Platform names are exempt, such as `XMLHttpRequest`.

**Source:** Google TypeScript Style Guide

### Name the key in map variables [Required]

The name of a `Map`, `Record`, or index-signature object must say what looks up what. Give index signature keys meaningful names too.

**Example:**

Avoid:

```ts
const prices = new Map<string, number>();
const users: Record<string, User> = {};
```

Prefer:

```ts
const priceBySku = new Map<string, number>();
const userById = new Map<string, User>();
```

**Source:** Team

## Import order

### Leave import order to tools [Recommended]

The order is: Node built-in modules, third-party packages, project aliases, relative paths. Write built-in modules with the `node:` prefix. Let Biome or an ESLint plugin sort automatically. Do not adjust by hand.

### Use `import type` for type-only imports [Recommended]

Under `verbatimModuleSyntax`, imports without `type` stay at runtime. With Node.js type stripping, this becomes a runtime error.

**Tooling:** `@typescript-eslint/consistent-type-imports`.

**Source:** TSConfig Reference, Node.js API Docs

### Prefer named exports [Recommended]

Default exports have no canonical name. Importers can name them anything, and typos are not reported. Exempt when a framework requires a default export.

**Tooling:** `import-x/no-default-export`.

**Source:** Google TypeScript Style Guide

## File organization

### Export only symbols used outside the module [Recommended]

Do not use `export let`. Access mutable state through functions. Do not write container classes with only static members: the file itself is the namespace. Do not use `namespace`.

**Source:** Google TypeScript Style Guide

### No circular dependencies [Required]

Circular dependencies cause initialization order problems and reads of uninitialized bindings. `export *` re-exporting everything is a common source and also slows builds, so do not use it. Module entry points that list exports explicitly are allowed, such as `export { createOrder } from './create-order.js'`.

**Tooling:** `import-x/no-cycle` or `madge --circular`.

**Source:** NestJS Docs
