# TypeScript / JavaScript project structure

## Directory structure

### Organize source by feature [Recommended]

`src/` holds source. `dist/` holds build output and is not committed. Keep a feature's routes, services, and types together. Do not spread code across the project by technical type, such as `controllers/` and `services/`. For Express, Fastify, and NestJS layouts, see "Organize feature directories by framework" in [node.md](node.md).

**Example:**

```text
src/
  orders/
    order-service.ts
    order-service.test.ts
    order-routes.ts
  config.ts
  server.ts
```

### Separate the app instance from listening [Recommended]

`app.ts` only builds the app and `server.ts` handles `listen`, so tests do not occupy a port.

**Source:** Fastify Docs

## Package management

### Use one package manager per project [Recommended]

Commit its lock file. Do not mix lock files from different package managers.

### Use frozen installs in CI [Recommended]

**Applies to:** team project

Use `npm ci` or `pnpm install --frozen-lockfile` in CI and image builds. When the lock file and `package.json` disagree, fail immediately and do not update the lock file.

**Source:** npm Docs, pnpm Docs

### Classify dependencies correctly [Recommended]

Put what the runtime needs in `dependencies`, and put type packages and build and test tools in `devDependencies`. Packages the code uses must be declared directly. Do not rely on transitive dependencies.

### Restrict dependency install scripts [Required]

**Applies to:** production service

In pnpm, list the dependencies allowed to run build scripts in `allowBuilds`. In npm, use `--ignore-scripts` or the `ignore-scripts` config.

**Source:** Node.js Security Best Practices, pnpm Docs

## Versions and pinning

### Declare the runtime version [Recommended]

Set `engines.node` in `package.json`. Use the same major version locally, in CI, and in images, and choose an LTS version. `engines` only produces warnings by default. Configure `engine-strict` to enforce it.

**Source:** npm Docs

### Keep the TypeScript version within toolchain requirements [Recommended]

Before upgrading TypeScript, check the version requirements of lint and other tools. When a tool does not support it yet, temporarily pin to a compatible version, record the condition for removing the pin, and upgrade once the tool supports it. Do not install two versions at once.

**Source:** typescript-eslint Docs

### Do not commit unused type packages [Recommended]

Once `types` lists global types explicitly, keep only the `@types/*` packages actually needed.

## Build

### Default to ESM in new projects [Recommended]

In Node.js projects, set `"type": "module"` in `package.json` and use `nodenext` for `module`.

| Item | ESM | CommonJS |
|---|---|---|
| Detection | `.mjs`, or `.js` under `"type": "module"` | `.cjs`, or `.js` under `"type": "commonjs"` |
| Relative imports | Extension required | Can be omitted |
| Current file path | `import.meta.dirname`, `import.meta.filename` | `__dirname`, `__filename` |
| Top-level `await` | Supported | Not supported |

**Exceptions:** For bundler projects, follow the bundler's docs on whether to set `"type": "module"`.

**Source:** Node.js API Docs, TypeScript Handbook

### Use the output file extension in relative imports under `nodenext` [Recommended]

When the source is `./order.ts`, write the import as `./order.js`. TypeScript does not rewrite ordinary import paths, and omitting the extension fails at runtime. When running `.ts` directly, write `./order.ts` and enable `allowImportingTsExtensions`. When also emitting JavaScript, enable `rewriteRelativeImportExtensions`.

**Source:** TypeScript Handbook, TSConfig Reference, Node.js API Docs

### Know the limits of `require(esm)` [Recommended]

Loading ESM with `require()` needs runtime support. The loaded module and its dependencies cannot use top-level `await`.

**Source:** Node.js API Docs

### Use `imports` for path aliases [Recommended]

`imports` in `package.json` is understood by both Node.js and TypeScript. `paths` in `tsconfig` does not rewrite output, so the runtime cannot find the modules, and under type stripping it fails outright. The `#/` prefix needs TypeScript 6.0 and Node.js 24.14+ or 25.4+.

**Example:**

```json
{
  "type": "module",
  "imports": {
    "#/*": "./dist/*"
  }
}
```

**Source:** Node.js API Docs, TypeScript Release Notes

### Type checking is a separate step [Recommended]

**Applies to:** team project

esbuild, tsx, and Node.js type stripping do not check types. CI must run `tsc --noEmit` separately, covering test files. When using these tools, enable `verbatimModuleSyntax` or `isolatedModules` so the compiler reports syntax that single-file transpilation cannot handle.

**Source:** TSConfig Reference, Node.js API Docs

### Use only erasable syntax when running `.ts` directly [Recommended]

Type stripping needs Node.js 22.18 or later. It does not read `tsconfig.json` and does not support `enum`, `namespace` with runtime code, or parameter properties. It suits scripts and tools. Enable `erasableSyntaxOnly` so the compiler reports errors early.

**Exceptions:** NestJS depends on `experimentalDecorators` and `emitDecoratorMetadata`, so it must be compiled and must not enable `erasableSyntaxOnly`.

**Source:** Node.js API Docs, TSConfig Reference, NestJS Docs

### Do not publish TypeScript source in dependency packages [Required]

Node.js refuses to process `.ts` files under `node_modules`. Publish compiled JavaScript and `.d.ts`.

**Source:** Node.js API Docs

### Declare public package entry points with `exports` [Recommended]

Libraries list entry points in `exports`. Unlisted paths cannot be imported. Control what gets published with the `files` allowlist, and check with `npm publish --dry-run` before publishing. Set `"private": true` on private packages.

**Source:** Node.js API Docs, Node.js Security Best Practices, npm Docs

## Multi-module or monorepo

### Manage multiple packages with workspaces [Recommended]

Use pnpm workspaces and write internal dependencies as `workspace:*` so they resolve to local packages. Unify shared dependency versions with `catalog:`.

**Source:** pnpm Docs

### Depend across packages only through entry points [Recommended]

Do not import across packages with relative paths, and do not reference another package's internal `src` files.

### Organize type checking with project references [Recommended]

Enable `composite` on referenced packages, list `references` in the root config, and build incrementally with `tsc -b`. This enforces the dependency direction between packages and shortens check time.

**Source:** TypeScript Handbook

### Keep one copy of shared config [Recommended]

Put `tsconfig.base.json` in the root. Each package inherits it with `extends` and writes only the differences. ESLint and formatter config also go in the root.
