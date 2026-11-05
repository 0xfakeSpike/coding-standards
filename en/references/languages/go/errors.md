# Go error handling

For language-independent principles, see [general/error-handling.md](../../general/error-handling.md). This document covers only Go's mechanisms.

## Error mechanics

### Return error last and check it immediately [Required]

Declare the return type as `error`, not a concrete error type. When `err != nil`, treat the other return values as invalid. Functions that take `context.Context` usually also return `error`.

**Source:** Google Go Style Guide

### Do not discard errors with _ [Required]

Handle the error or return it. When it can truly be ignored, add a one-line comment next to it with the reason. `Close` on read-only resources and `defer tx.Rollback()` may go unchecked. After a commit, `Rollback` returns `sql.ErrTxDone`. On write paths, check as described in [idioms.md](idioms.md).

**Tooling:** `errcheck`

**Source:** Go Code Review Comments, Google Go Style Guide

### Do not panic for ordinary errors [Required]

Use `panic` only for program defects that make continuing impossible, or for fatal failures during startup. Use `MustXxx` only for package-level variables and startup initialization, not for user input or request handling. `recover` is not an error-handling mechanism. Use it only at request boundaries, background job entry points, and tool execution entry points.

**Source:** Effective Go, Go Code Review Comments, Uber Go Style Guide

### Exit the process only in main [Recommended]

Call `os.Exit` and `log.Fatal` only in `main`, and only once. They skip `defer`. Put all other logic in a `run` that returns `error`.

**Source:** Uber Go Style Guide

### Start error messages lowercase, with no ending punctuation [Recommended]

Error messages are usually embedded in longer messages. Keep proper nouns and abbreviations as they are. This rule does not apply to logs.

**Tooling:** `staticcheck`'s ST1005

**Source:** Go Code Review Comments, Google Go Style Guide

## Custom errors and ownership

### Define the errors each package needs in that package [Required]

Define the errors a package returns inside that package. Do not create an `errs`, `errors`, or `common` error package shared by all packages. When an upper layer needs to tell errors apart, reference the lower package's errors, or define its own in its package and convert.

**Example:**

Avoid:

```go
package errs

var (
	ErrOrderNotFound = errors.New("order not found")
	ErrUserNotFound  = errors.New("user not found")
)
```

Prefer:

```go
package order

var ErrNotFound = errors.New("order not found")

package user

var ErrNotFound = errors.New("user not found")
```

**Source:** Team

### Choose the error form by whether callers need to check it [Recommended]

| Caller needs to check | Message | Form |
|---|---|---|
| No | Static | `errors.New` |
| No | Dynamic | `fmt.Errorf` |
| Yes | Static | Package-level `var ErrXxx = errors.New(...)` |
| Yes | Dynamic | Custom type `XxxError` |

Do not export errors that no caller checks. Exported error variables and types are part of the package's API.

**Source:** Uber Go Style Guide, Google Go Style Guide

### Put structured fields on errors [Recommended]

When callers need data from an error, put it in fields of a custom type, such as the path in `os.PathError`. Do not make callers parse strings. Implement `Unwrap` when wrapping a lower-level error.

**Source:** Google Go Style Guide, Go Blog: Error Handling

## Wrapping and checking

### Use %w when adding context [Required]

State the action in progress and its target. Put `%w` at the end so the text order matches the error chain.

**Example:**

```go
if err := gateway.Charge(ctx, order); err != nil {
	return fmt.Errorf("charge order %s: %w", order.ID, err)
}
```

**Exceptions:** A sentinel error used as a category label may go first, written as `%w: details`.

**Tooling:** `errorlint`

**Source:** Google Go Style Guide, Uber Go Style Guide, Go Release Notes

### Keep filler out of context [Recommended]

Do not write `failed to` or `unable to`, or they pile up layer by layer. Do not repeat what the lower layer already says: the `os.Open` error already includes the path. When there is nothing new to add, `return err` directly.

**Source:** Google Go Style Guide, Uber Go Style Guide

### Do not let implementation-detail errors cross package boundaries [Recommended]

Wrapping with `%w` makes the lower-level error part of your API. Convert cases callers need to check into this package's errors, so callers do not depend on implementation details such as `sql.ErrNoRows`. Use `%v` to hide all lower-level errors.

**Example:**

Avoid:

```go
func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).Scan(&o.ID, &o.TotalCents)
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}
```

Prefer:

```go
func (s *Store) ByID(ctx context.Context, id int64) (Order, error) {
	var o Order
	err := s.db.QueryRowContext(ctx, orderByIDSQL, id).Scan(&o.ID, &o.TotalCents)
	if errors.Is(err, sql.ErrNoRows) {
		return Order{}, ErrNotFound
	}
	if err != nil {
		return Order{}, fmt.Errorf("order by id %d: %w", id, err)
	}
	return o, nil
}
```

**Source:** Go Blog: Error Handling

### Check with errors.Is and errors.As [Required]

Do not use `==` on errors that may be wrapped, and do not compare error strings. Go 1.26+ can use the type-safe `errors.AsType`. Combine multiple errors with `errors.Join` (Go 1.20+).

**Example:**

```go
if errors.Is(err, order.ErrNotFound) {
	return nil
}
if ve, ok := errors.AsType[*ValidationError](err); ok {
	return badField(ve.Field)
}
```

**Tooling:** `errorlint`

**Source:** Go Blog: Error Handling, Google Go Style Guide, Go Release Notes

## Logging

### Log errors with slog at request boundaries [Recommended]

Either log or return, not both. For the principle, see [general/error-handling.md](../../general/error-handling.md). Log at the request boundary or background job entry point, and pass the error as a field through `log/slog`.

**Source:** Uber Go Style Guide, Google Go Style Guide

## Framework boundaries

### Map errors in one place at the transport layer [Recommended]

Business packages do not reference HTTP status codes or gRPC `codes`. Handlers use `errors.Is` and `errors.As` to map package errors to responses. For requirements on externally visible messages, see [web-services.md](web-services.md).

### Catch panics in long-running goroutines [Required]

**Applies to:** production service

`net/http` recovers panics only in the handler's goroutine. A panic in a goroutine you started takes down the whole process. In long-running background jobs and tool execution entry points, catch panics with `recover`, convert them to errors, and log them. For gRPC requirements, see [web-services.md](web-services.md).

**Source:** Go Standard Library Docs
