# Frontend architecture

This document covers directory and module boundaries, rendering modes, monorepo, micro frontends, and the design system. For internal component design, see [components.md](components.md). For language-independent responsibility rules, see [responsibilities.md](../general/responsibilities.md).

## Directory and module boundaries

### Keep responsibilities and layers clear [Required]

Every directory, module, and component must be able to state which layer it belongs to and what it is responsible for. Route entries only assemble pages, feature modules own business flows, and the shared layer holds no business knowledge.

**Why:** Once layers mix, one change ripples to many places, and you cannot tell where new code belongs.

**Source:** Team

### Organize code by business feature [Recommended]

Keep a feature's components, logic, requests, and types together. Do not scatter them across global directories by file type. Vue projects use `composables/` instead of `hooks/`.

```text
src/
├── app/
├── features/
│   └── order/
│       ├── components/
│       ├── hooks/
│       ├── api.ts
│       └── types.ts
├── components/
├── hooks/
├── lib/
└── utils/
```

| Directory | Contents |
|---|---|
| `app/` or `pages/` | Route entries |
| `features/` | Business features |
| `components/`, `hooks/` | Shared UI and logic across features |
| `lib/` | Request client, third-party wrappers |
| `utils/` | Pure functions with no business meaning |

**Why:** Changing a feature does not span multiple directories, and deleting a feature removes it as a whole.

### Keep dependencies one-way [Recommended]

Route entries may depend on `features`, and `features` may depend on `components`, `lib`, and `utils`. The reverse is not allowed. Dependencies between features go through the public entry point described in the next rule. The shared layer must not import business modules.

**Why:** Reverse dependencies put business knowledge into shared code and easily create circular dependencies.

**Tooling:** ESLint `no-restricted-imports`. Nuxt auto-imports do not go through `import` statements, so this check does not cover them. For the directory approach, see [nuxt.md](nuxt.md).

### Use only public entry points between features [Recommended]

When one feature uses another, import only modules the other explicitly exposes, such as `@/features/order`. Do not reach into internal files such as `@/features/order/components/table/format`. Entry files list exports one by one. Do not use `export *`.

**Why:** Once external code references an internal file, it can no longer be refactored freely.

### Keep one copy of each thing [Required]

Things such as date formats, money formats, breakpoint constants, and the request client have one implementation across the project. For example, all dates call the same `formatDate`. Do not write format strings separately in each place.

**Why:** Multiple implementations eventually diverge, and changes easily miss one.

**Source:** Team

## Rendering modes

### Choose the rendering mode per page [Recommended]

Measure the bottleneck before choosing, and consider static or server-side rendering first. One site can mix modes by page.

| Mode | Best for | Cost |
|---|---|---|
| CSR | Highly interactive apps behind login | First paint depends on JavaScript, and search engine indexing needs extra verification |
| SSR | Content that varies per request and needs a fast first paint | Consumes server compute, time to first byte may increase, and caching must be considered |
| SSG | Content fixed at build time | All URLs must be known in advance. Not suited to massive or personalized page sets |
| ISR | Large content volumes where brief staleness is acceptable | Depends on the framework and deployment platform, and an invalidation strategy must be defined |

**Source:** web.dev: Rendering Patterns, Next.js Docs, Nuxt Docs

### Do not use pure client-side rendering for content pages [Recommended]

For pages where first-paint speed or search indexing matters, output complete HTML with SSR or SSG.

**Why:** Pure CSR has no content until JavaScript downloads and runs. SSR must also control client code size, or the page looks ready but does not respond to input.

**Exceptions:** Admin and tool pages accessible only after login.

**Source:** web.dev: Rendering Patterns, Vue Docs

### Do not share module-level mutable state on the server [Required]

During server-side rendering, a module initializes once and is reused by all requests. User-related state must be created per request and must not live in module-level variables.

**Why:** Module-level state leaks one user's data to another.

**Source:** Vue Docs, Nuxt Docs

### Make the first render match on server and client [Required]

With server-side rendering, access browser APIs such as `window` and `localStorage` only after mount or in client-only code. Random numbers, the current time, and the local time zone must not make the two sides render different content. Prefer CSS media queries for responsive differences.

**Why:** Hydration mismatches add performance overhead and produce a wrong UI.

**Source:** Vue Docs, Nuxt Docs

## Monorepo

### Separate apps and shared packages [Recommended]

`apps/` holds apps and services, and `packages/` holds shared libraries and tool configuration. Do not nest a package inside another package.

**Source:** Turborepo Docs

### Install dependencies in the package that uses them [Recommended]

Each package's `package.json` lists every dependency it uses. The root holds only tools that manage the repository.

**Why:** Dependency relationships stay clear, and each package's build cache and pruning stay accurate.

**Source:** Turborepo Docs

For workspaces, internal dependencies, and cross-package imports, see [project.md](../languages/typescript/project.md).

### Declare dependencies and outputs for build tasks [Recommended]

Use `dependsOn: ["^build"]` to ensure dependency packages build first, and declare build outputs with `outputs`. Otherwise Turborepo caches no files. Disable caching for tasks with side effects, such as deployment.

**Source:** Turborepo Docs

## Micro frontends

### Do not use micro frontends by default [Recommended]

Adopt them only when multiple teams truly need independent development and independent deployment. Integrate at runtime, and do not merge sub-apps into dependency packages at build time. Sub-apps communicate only through URLs, custom events, or callbacks passed in by the container, and keep state and styles isolated from each other.

**Why:** They bring duplicate dependencies, environment differences, and more operations work. Build-time integration also requires synchronized releases, which defeats independent deployment.

**Source:** Fowler: Micro Frontends

## Design system

For rules on visual values, see [css.md](css.md).

### Keep only generic UI in the shared library, and extract after validation [Recommended]

The design system includes only components with no business meaning, such as icons, buttons, and inputs. Implement inside a feature first, and extract once real duplication appears and the API is stable.

**Why:** Extracting too early makes it hard to guess the API correctly, and business components in a shared library couple all consumers.

**Source:** Fowler: Micro Frontends
