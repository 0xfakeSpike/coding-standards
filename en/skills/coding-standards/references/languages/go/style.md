# Go style

Do not rely on human review for formatting that tools can handle.

## Tool configuration

### Format with gofmt and goimports [Recommended]

**Applies to:** team project

Format before committing. Do not debate indentation, alignment, or import grouping.

**Tooling:** `gofmt -l .`, `goimports -l .`

**Source:** Go Code Review Comments, Effective Go

### Gate on static checks [Recommended]

**Applies to:** team project

CI runs `go vet ./...` and `golangci-lint run`, and the latter includes at least `staticcheck`. On Go 1.26+, `go fix` can bulk-update code to newer forms. Commit those changes separately.

**Source:** Go Security Best Practices, Go Release Notes

### Disable doc comment rules [Required]

Go convention requires doc comments on all exported identifiers, but the team rule does not enforce this, and the team rule wins. Externally published libraries keep doc comments and these checks. Disable `revive`'s `exported` and `package-comments`, and do not enable `staticcheck`'s ST1000, ST1020, ST1021, and ST1022. Remove `golint` outright from projects still using it and replace it with `revive`. Explicitly configuring any `revive` rule disables the default rules, so also turn on `enable-default-rules`. For configuration options, follow the docs for the golangci-lint version in use.

**Example:**

```yaml
version: "2"
linters:
  enable:
    - bodyclose
    - errorlint
    - gosec
    - noctx
    - revive
    - sqlclosecheck
  settings:
    revive:
      enable-default-rules: true
      rules:
        - name: exported
          disabled: true
        - name: package-comments
          disabled: true
  exclusions:
    presets:
      - comments
      - std-error-handling
formatters:
  enable:
    - gofmt
    - goimports
```

**Tooling:** Put the `//nolint` reason after the directive, such as `//nolint:gosec // path comes from internal config`.

**Source:** Team, golangci-lint Docs, revive Docs, staticcheck Docs

## Formatting

### Comments record only what the code cannot say [Required]

Do not write comments that restate the code, including doc comments that only repeat the name and signature. API docs and license headers of externally published libraries are excepted. Tool directives are also excepted, such as `//go:build`, `//go:generate`, `//go:embed`, `//nolint`, and `// Deprecated:`. When the code cannot express a reason, write it in one line next to the code, such as an external system limit, counter-intuitive behavior, or a single-point trade-off like ignoring an error, a type assertion, or suppressing lint. When an exported identifier has a constraint its name and signature cannot express, put it in the doc comment. Put notes that span several places in the code in project docs.

**Source:** Team

### Break lines by meaning [Recommended]

Go has no fixed line width. When a line is too long, change names and structure first. Do not mechanically wrap in the middle of a call.

**Source:** Go Code Review Comments

### Keep the normal path unindented [Recommended]

Handle failure branches first and return. When a branch ends with `return` or `continue`, do not write `else`. When a variable is used after the `if`, declare it on its own line.

**Source:** Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide

## Naming

| Item | Rule | Example |
|---|---|---|
| Package | Short lowercase word, no underscores or camelCase, not plural | `billing`, `tabwriter` |
| Exported identifier | `MixedCaps` | `OrderService` |
| Unexported identifier | `mixedCaps` | `orderTotal` |
| Constant | Same as above, no all caps, named by purpose | `MaxRetryCount` |
| Initialism | Keep case consistent | `userID`, `HTTPServer`, `parseURL` |
| Error variable | Starts with `Err` or `err` | `ErrNotFound` |
| Error type | Ends with `Error` | `ValidationError` |
| Single-method interface | Method name plus `er` | `Reader`, `Stringer` |
| File | `snake_case.go` | `order_service.go` |

**Source:** Effective Go, Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide

### Name the key in map variables [Required]

Write the name as `valueByKey`, or use a named type as the key, so readers know what it is indexed by without looking at the assignment.

**Example:**

Avoid:

```go
var cache map[string]Order
prices := map[string]int64{}
```

Prefer:

```go
var orderByID map[OrderID]Order
priceBySKU := map[string]int64{}
```

**Source:** Team

### Do not repeat the package name or type in names [Recommended]

Callers write `order.Service`, so do not name it `order.OrderService`. `widget.New` beats `widget.NewWidget`. Do not put type words in variable names: write `users`, not `userSlice`.

**Source:** Go Code Review Comments, Google Go Style Guide

### Do not prefix getters with Get [Recommended]

The getter for field `owner` is `Owner()`, and the setter is `SetOwner()`. For remote calls or expensive computation, use `Fetch` or `Compute` to signal the cost.

**Source:** Effective Go, Google Go Style Guide

### Abbreviate the type for receivers [Recommended]

Use one or two letters, consistent across all methods of a type. Do not use `self`, `this`, or `me`.

**Source:** Go Code Review Comments, Google Go Style Guide

### Do not use uninformative package names [Recommended]

Do not use `util`, `common`, `helpers`, `misc`, `types`, or `interfaces`. Do not use `model` as a package name either. Put types in their owning business package. See [project.md](project.md).

**Source:** Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide

### Match name length to scope [Recommended]

Use short names such as `i` in loops, `r` for readers, and `w` for writers. The farther from the declaration, the fuller the name. Do not abbreviate by dropping letters: `Sandbox` beats `Sbx`.

**Source:** Go Code Review Comments, Google Go Style Guide

### Do not shadow predeclared identifiers or common package names [Recommended]

Do not name variables `error`, `string`, or `len`, and in larger scopes do not name them `url` or `time`.

**Source:** Uber Go Style Guide, Google Go Style Guide

## Import order

### Put the standard library in the first group [Recommended]

Use one group for the standard library and one for everything else, with a blank line between. When a third group is needed, standardize it with `goimports -local`.

**Source:** Go Code Review Comments, Uber Go Style Guide

### Rename imports only on conflict [Recommended]

Add an alias only when the package name differs from the last path segment. On conflict, rename the local or in-project package. For packages generated from proto, use the full word with a `pb` suffix, such as `orderpb`.

**Source:** Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide

### Avoid dot imports and keep blank imports at entry points [Recommended]

Do not write `import . "pkg"`. `import _ "pkg"` appears only in `main` packages or tests.

**Exceptions:** Tests that can live only in a `foo_test` package because of an import cycle may dot-import the package under test.

**Source:** Go Code Review Comments, Google Go Style Guide

## File organization

### Order code for reading [Recommended]

Put types and constants first, then constructors and the type's methods, with plain helper functions last. Group methods by receiver, roughly in call order. Go has no one-type-per-file convention. Keep closely related types together.

**Source:** Uber Go Style Guide, Google Go Style Guide

### Avoid init functions [Recommended]

In `init`, do no I/O, do not read environment variables or the working directory, do not start goroutines, and do not depend on the order of other `init` functions. Move this work to `main` or to explicitly called functions.

**Exceptions:** Registration hooks such as `database/sql` drivers.

**Source:** Uber Go Style Guide
