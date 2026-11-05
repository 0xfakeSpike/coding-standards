# Go project

## Directory structure

### Start with the simplest layout [Recommended]

In a small project, keep all code in the root directory. Add layers when there are multiple executables or supporting packages. Do not force `pkg/` just for tidy directories.

**Example:**

With two executables and two business packages:

```text
go.mod
cmd/
  api/
    main.go
  worker/
    main.go
internal/
  order/
    order.go
    store.go
  payment/
    payment.go
    store.go
migrations/
```

**Source:** Go Module Layout

### Put executables in cmd and internal code in internal [Recommended]

`cmd/<name>/main.go` only parses configuration, wires dependencies, and starts. Supporting packages go in `internal/` by default. Put them outside only when other modules truly need them.

**Why:** The Go toolchain enforces `internal`. External modules cannot import it, so the internal structure can change freely.

**Source:** Go Module Layout

### Organize packages by business responsibility [Recommended]

A package name says what it provides, such as `order` and `payment`. Avoid packages split by kind, such as `controllers` and `services`. Otherwise one change spans several packages, and import cycles come easily. Keep tightly coupled types in the same package.

**Source:** Google Go Style Guide

### Give each table a single entry point [Required]

Each table belongs to one business package. Keep the table's type in that package and put all reads and writes for the table in the package's `store.go`. Business code does not write SQL directly. Other packages that need this data call the owning package's exported methods and do not write SQL for this table. For the read-only JOIN exception, see below. View all table schemas together in `migrations/`. When all code is in one package, that package is the owner.

`DBTX` and `WithTx(*sql.Tx)` are `database/sql` examples, not structural requirements. Other drivers keep the same structure: with pgx, the interface lists `Exec` and `QueryRow`, and `WithTx` takes `pgx.Tx`; with GORM, `Store` holds a `*gorm.DB`, and `WithTx` passes in the transaction's `*gorm.DB`; with sqlc, use the generated `Queries.WithTx`. For exact signatures, follow the docs for the version in use.

**Why:** Changing a table then means looking only at the owning package and read-only JOIN packages, and the method list on `Store` shows every use of the table.

**Example:**

```go
package order

const (
	orderByIDSQL = "SELECT id, user_id, total_cents, created_at FROM orders WHERE id = ?"
	markPaidSQL  = "UPDATE orders SET status = 'paid' WHERE id = ? AND status = 'pending'"
)

var (
	ErrNotFound   = errors.New("order not found")
	ErrNotPending = errors.New("order not pending")
)

type Order struct {
	ID         int64
	UserID     int64
	TotalCents int64
	CreatedAt  time.Time
}

type DBTX interface {
	ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
	QueryRowContext(ctx context.Context, query string, args ...any) *sql.Row
}

type Store struct {
	db DBTX
}

func NewStore(db DBTX) *Store {
	return &Store{db: db}
}

func (s *Store) WithTx(tx *sql.Tx) *Store {
	return &Store{db: tx}
}

func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).
		Scan(&o.ID, &o.UserID, &o.TotalCents, &o.CreatedAt)
	if errors.Is(err, sql.ErrNoRows) {
		return Order{}, ErrNotFound
	}
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}

func (s *Store) MarkPaid(ctx context.Context, id int64) error {
	res, err := s.db.ExecContext(ctx, markPaidSQL, id)
	if err != nil {
		return fmt.Errorf("mark order %d paid: %w", id, err)
	}
	n, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("mark order %d paid: %w", id, err)
	}
	if n == 0 {
		return ErrNotPending
	}
	return nil
}
```

**Source:** Team

## Across business packages

The rules in this section assume the project has two or more business packages.

### Put cross-business flows in an upper-level business package [Recommended]

Business packages may import each other in one direction, with the direction set by domain dependencies. For example, when `invoice` needs order data, it calls `order`'s exported methods directly. Create a new upper-level package named after the flow, such as `internal/checkout`, only when a flow must coordinate packages that do not depend on each other, or when both sides need each other. The upper-level package depends on `order` and `payment`. Those two do not import each other, and SQL stays in their own `store.go`.

**Why:** Go does not allow import cycles, and a flow should not be scattered across packages.

**Example:**

```text
internal/
  checkout/
    checkout.go
  order/
  payment/
```

### Let the orchestrating package start cross-business transactions [Recommended]

When tables that need atomic writes belong to several business packages, the upper-level package starts the transaction and passes it to each `Store` through `WithTx`. `Store` depends on the `DBTX` interface, which both `*sql.DB` and `*sql.Tx` satisfy. Business packages do not start cross-package transactions, and code inside a transaction does not call external services.

For multi-table transactions within one package, a business function in that package starts the transaction, written the same way. `DBTX` has no `BeginTx`, so the type that starts transactions holds a separate `*sql.DB`. `*sql.DB` and `*sql.Tx` appear only in these types and in `Store`.

First claim the state with a conditional update, then write to other tables. Duplicate or concurrent requests get `ErrNotPending` at this step and do not write a second payment record.

**Example:**

```go
package checkout

type Service struct {
	db       *sql.DB
	orders   *order.Store
	payments *payment.Store
}

func (s *Service) Pay(ctx context.Context, orderID int64) error {
	tx, err := s.db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin pay tx: %w", err)
	}
	defer tx.Rollback()

	orders := s.orders.WithTx(tx)
	o, err := orders.ByID(ctx, orderID)
	if err != nil {
		return err
	}
	if err := orders.MarkPaid(ctx, o.ID); err != nil {
		return err
	}
	if err := s.payments.WithTx(tx).Create(ctx, o.ID, o.TotalCents); err != nil {
		return err
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("commit pay tx: %w", err)
	}
	return nil
}
```

### Prefer separate calls for cross-business reads [Recommended]

When data volume is small, call the two `Store`s separately and combine the results in the upper-level package.

### Put read-only queries that need a JOIN in a dedicated package [Recommended]

When a cross-business list or report truly needs a JOIN, put it in a dedicated read-only package, such as `internal/orderreport`. This is the only exception to "do not write SQL for another package's tables". These packages only query and never write.

**Why:** Putting the JOIN in either business package makes it depend on the other business's table structure.

## Package boundaries

### Consider merging packages whose tables always change together [Recommended]

When two packages' tables are always modified and queried together, the boundary may be wrong. Consider merging them into one package.

### Avoid import cycles by adjusting package boundaries [Recommended]

When an import cycle appears, check the package split first, and move shared parts down or merge them. Forcing a break with an interface usually means the package structure is wrong.

**Source:** Google Go Style Guide

## Package management

### Commit go.mod and go.sum [Required]

Both files go into version control. `go.sum` verifies the integrity of downloaded content.

**Source:** Go Dependency Management

### Check dependency tidiness in CI [Recommended]

After `go mod tidy`, `go.mod` and `go.sum` must show no diff. Delete unused dependencies outright.

**Tooling:** `go mod tidy -diff`

**Source:** Go Dependency Management

### Manage development tools with the tool directive [Recommended]

On Go 1.24+, register tools such as `stringer` with `go get -tool` and run them with `go tool stringer`. Their versions are locked by `go.mod`.

**Source:** Go Dependency Management, Go Modules Reference

### Keep replace out of released versions [Recommended]

A module that others depend on must not rely on `replace`. It takes effect only in the main module, and dependents ignore it. For local work across several modules, prefer `go.work`.

**Exceptions:** To fix an unreleased upstream defect, you may temporarily point to a fork. Put the reason in a comment next to `replace`.

**Source:** Go Modules Reference

### Configure GOPRIVATE for private modules [Recommended]

Set `GOPRIVATE` for private modules, so they download directly from the source repository and module paths are not sent to the public proxy.

**Source:** Go Modules Reference

## Versions and locking

### Set the go directive to the real minimum version [Recommended]

The `go` directive is an enforced minimum version. It also determines language features and semantics such as loop variables. Raising it changes behavior, so commit it separately. `toolchain` takes effect only in the main module and suggests a specific toolchain version.

**Source:** Go Modules Reference

### Let go.mod lock versions and upgrade deliberately [Recommended]

Go uses minimal version selection and does not pick the latest version automatically. Upgrade with `go get example.com/pkg@v1.2.3`. Do not mix an unrelated `go get -u ./...` into feature commits. After upgrading, run tests and `govulncheck`.

**Source:** Go Modules Reference, Go Security Best Practices

### Keep Go patch versions current [Recommended]

**Applies to:** production service

Security fixes ship in patch releases. Keep the Go version used in CI and images up to date, and consistent with local.

**Source:** Go Security Best Practices

## Build

### Do not let builds rewrite go.mod [Recommended]

When `go.mod` needs changes, the build fails outright. This is the default behavior. Do not use `-mod=mod` in CI.

**Source:** Go Modules Reference

### Use reproducible flags for release builds [Recommended]

Use `go build -trimpath` for releases to strip local paths. Inject version info with `-ldflags "-X main.version=$VERSION"`, or read embedded VCS info with `go version -m`. Set `CGO_ENABLED=0` for services that do not depend on C libraries.

**Example:**

```sh
CGO_ENABLED=0 go build -trimpath -o bin/api ./cmd/api
```

### Use build constraints for platform differences [Recommended]

Use `//go:build linux` or a `_linux.go` suffix, not runtime checks. Embed static assets in the binary with `//go:embed`.

### Give generated code a fixed entry point [Recommended]

Declare generation commands with `//go:generate`. The project decides once whether generated output is committed. When it is committed, CI regenerates it and checks for no diff.

## Multi-module or monorepo

### Prefer a single module [Recommended]

Use one `go.mod` per repository, with boundaries drawn by `internal/` and directories. Split modules only when release cycles are truly independent.

**Source:** Go Module Layout

### Use go.work for local multi-module work [Recommended]

Usually do not commit `go.work`. Otherwise it may override developers' own workspaces and make CI test the wrong dependency versions. Set `GOWORK=off` in CI, or commit it only when modules are developed together as a set.

**Source:** Go Modules Reference
