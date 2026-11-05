# Build and browser compatibility

This document uses Vite as the example and covers build configuration, environment variables, and browser support. Other build tools follow the same requirements. For build reproducibility and the release process, see [Release](../general/release.md). For loading performance, see [Performance](performance.md). For Source Maps, see [Security](security.md). For TypeScript configuration, see [TypeScript project configuration](../languages/typescript/project.md).

## Build configuration

### Type-check Vue projects with `vue-tsc` [Recommended]

Vite only transpiles `.ts` files and does not type-check them. For type-checking and single-file transpilation configuration, see [project.md](../languages/typescript/project.md). Vue projects use `vue-tsc --noEmit`.

**Source:** Vite Docs

### Use `import.meta.env.BASE_URL` for dynamic paths [Recommended]

Configure `base` when deploying to a subpath. Use `import.meta.env.BASE_URL` to build asset URLs at runtime. This variable is statically replaced at build time, and the form `import.meta.env['BASE_URL']` is not replaced.

**Source:** Vite Docs

For Source Map configuration, see [security.md](security.md).

## Environment variables

### Treat frontend environment variables as public [Required]

`VITE_*` and `NEXT_PUBLIC_*` are all bundled into the build output. Never put secrets in them. Do not set `envPrefix` to an empty string. `.env.*.local` stores local values only, and `.gitignore` must include `*.local`.

**Source:** Vite Docs

### Declare variable types and convert values [Recommended]

Extend `ImportMetaEnv` in `vite-env.d.ts`. Environment variable values are all strings, so convert numbers and booleans before use.

Avoid:

```ts
if (import.meta.env.VITE_ENABLE_CHAT) {
  startChat();
}
```

Prefer:

```ts
if (import.meta.env.VITE_ENABLE_CHAT === 'true') {
  startChat();
}
```

**Why:** The string `'false'` is also truthy.

**Source:** Vite Docs

### Treat public variables as fixed at build time [Recommended]

Environment variables are written into the build output at build time, so changing them requires a rebuild. When the same build is deployed to multiple environments, have the server deliver configuration.

**Source:** Vite Docs, Next.js Docs

## Code splitting

### Split with dynamic imports [Recommended]

For route splitting, see [Routing](routing.md). For large dependencies, see [Performance](performance.md). When the framework provides split configuration, use the framework's approach. When you need manual chunking, use the build tool's official option, such as Vite's `build.rolldownOptions.output.codeSplitting`, which requires Vite 8 or later.

**Source:** Vite Docs

### Do not raise the warning threshold to hide large files [Recommended]

When `build.chunkSizeWarningLimit` warns, analyze and split the output first. This limit compares uncompressed size, because script size relates to execution time.

**Source:** Vite Docs

## Browser support

### Declare supported browsers explicitly [Recommended]

Write the support range as Browserslist configuration in `.browserslistrc` or `package.json`. When the range is unclear, start from `defaults`.

```text
defaults and fully supports es6-module
```

**Why:** Without a declaration, each tool uses its own default, and the output does not match expectations.

**Source:** Browserslist Docs

### Do not define the range with a single condition [Recommended]

When you need to adjust the range, combine conditions, such as `last 2 versions, not dead, > 0.2%`. `last n versions` alone misses older versions that are still popular. Name browsers directly only for fixed environments. Update data with `update-browserslist-db`, and confirm the actual range with `npx browserslist`.

**Source:** Browserslist Docs

### Keep the build target consistent with the declaration [Recommended]

Vite's `build.target` does not read Browserslist. The default target is Baseline Widely Available, pinned at each Vite release. When the declared range is below the default target, set `build.target` explicitly.

```ts
export default defineConfig({
  build: { target: ['chrome100', 'safari15', 'firefox100'] },
});
```

**Source:** Vite Docs

### Use feature detection, not User-Agent checks [Recommended]

In JavaScript, check whether the capability exists. In CSS, use `@supports`.

Avoid:

```ts
if (navigator.userAgent.includes('Safari')) {
  useFallbackShare();
}
```

Prefer:

```ts
if ('share' in navigator) {
  await navigator.share(data);
}
```

**Source:** MDN

## Polyfills

### Do not treat syntax transforms as polyfills [Required]

By default, Vite only transforms syntax and does not inject polyfills. When you use an API the target browsers lack, provide a polyfill explicitly.

**Source:** Vite Docs

### Import polyfills on demand per target browser [Recommended]

Babel projects inject core-js on demand. Follow the docs for your Babel version for the configuration. Specify the minor version in `corejs`. With only the major version, modules added in minor versions are not injected. Do not import core-js by hand in this mode. Libraries published for others must not pollute globals, so use `core-js-pure`.

**Source:** core-js Docs

### Use the official plugin for legacy browsers [Recommended]

When you need to support browsers without native ESM, use `@vitejs/plugin-legacy` to generate separate output, loaded only in those browsers. When `targets` is not set, it reads the Browserslist configuration. When `modernTargets` is not set, do not use `modernPolyfills: true`, because auto-detection injects too much. The plugin relies on inline scripts, so under a strict CSP, use the exported `cspHashes` to generate the response header.

**Source:** Vite Docs

## Release caching

### Upload assets before switching HTML on deploy [Recommended]

**Applies to:** production service

Keep the content hash in output file names. For response headers, see [Performance](performance.md). On deploy, upload static assets first, then switch HTML.

**Why:** If HTML switches first, assets the new page references may not exist yet.

**Source:** web.dev: HTTP Cache

For handling old chunk load failures after a release, see [Routing](routing.md).
