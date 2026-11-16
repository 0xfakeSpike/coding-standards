# Go design patterns

This file covers the idiomatic forms of design patterns in Go. For when to use which pattern, see [design-patterns.md](../../general/design-patterns.md). Go has no classes or inheritance, so copied Java-style structures look most out of place in Go. For interfaces, see "Define interfaces on the consumer side" and "Do not define interfaces early" in [types.md](types.md). For optional parameters, see "Use a config struct or functional options for optional parameters".

## Structure

### No singletons: create dependencies in `main` and pass them in [Recommended]

Do not write a global instance with `GetInstance()` and `sync.Once`. Create objects such as connection pools and clients once, in `main` or in the function that wires dependencies, and pass them to their users through constructors, see "Do not use mutable package-level variables" in [idioms.md](idioms.md). Use `sync.Once` only for lazy initialization inside a package, and do not expose a global instance.

**Source:** Uber Go Style Guide, Google Go Style Guide

### Use `New` functions for construction, not factory types or builders [Recommended]

A constructor is a plain `NewX` function that returns a concrete type. When the concrete type is decided only at run time, the factory function returns an interface, as in the exceptions to "Define interfaces on the consumer side" in [types.md](types.md). Do not write `XxxFactory` structs or chained builders.

### Do not use embedding to imitate inheritance [Recommended]

A Java-style template method lets subclasses override the parent's steps, but embedding in Go is not inheritance: methods of the embedded type cannot call the outer type's method of the same name, so the "override" never takes effect. When flows are the same and some steps differ, pass the differing steps in as functions or small interfaces. For embedding rules, see "Embed with care" in [types.md](types.md).

**Example:**

```go
type exporter struct {
	format func(rows []Row) ([]byte, error)
}

func (e exporter) Export(ctx context.Context, q Query) error {
	rows, err := query(ctx, q)
	if err != nil {
		return fmt.Errorf("query rows: %w", err)
	}
	data, err := e.format(rows)
	if err != nil {
		return fmt.Errorf("format rows: %w", err)
	}
	return upload(ctx, data)
}
```

**Source:** Effective Go

### No `Get` prefix on getters, and no methods that only pass a field through [Recommended]

The read method for field `owner` is `Owner()`, and the write method is `SetOwner()`. When there is no invariant to protect, export the field instead of writing methods that only pass it through.

**Source:** Effective Go

## Behavior

### Use function types for strategies [Recommended]

Write one function per case, and one function that returns the matching implementation for a given type. With the `exhaustive` check, a `switch` points out every place that misses a new type, see "Use a named type with iota, starting at 1" in [types.md](types.md). When a strategy needs several methods, the consumer defines a small interface.

**Example:**

```go
type exportFunc func(rows []Row) ([]byte, error)

func exporterFor(f Format) (exportFunc, error) {
	switch f {
	case FormatCSV:
		return toCSV, nil
	case FormatXLSX:
		return toXLSX, nil
	default:
		return nil, fmt.Errorf("unsupported export format %v", f)
	}
}
```

### Use a named type and a `switch` for state machines [Recommended]

Express states with a named integer type and `iota`. The transition method does a `switch` on the state, and each branch sets the next state only after its step has finished without error. An unknown state returns an error.

**Example:**

```go
type step int

const (
	stepAwaitingToken step = iota + 1
	stepUploading
	stepMerging
	stepDone
)

func (u *upload) advance(ctx context.Context) error {
	switch u.step {
	case stepAwaitingToken:
		token, err := fetchToken(ctx)
		if err != nil {
			return fmt.Errorf("fetch token: %w", err)
		}
		u.token, u.step = token, stepUploading
	case stepUploading:
		parts, err := uploadParts(ctx, u.token)
		if err != nil {
			return fmt.Errorf("upload parts: %w", err)
		}
		u.parts, u.step = parts, stepMerging
	case stepMerging:
		fileID, err := mergeParts(ctx, u.token, u.parts)
		if err != nil {
			return fmt.Errorf("merge parts: %w", err)
		}
		u.fileID, u.step = fileID, stepDone
	case stepDone:
	default:
		return fmt.Errorf("unknown upload step %d", u.step)
	}
	return nil
}
```

### Use wrapper functions for cross-cutting logic [Recommended]

Write HTTP cross-cutting logic as `func(http.Handler) http.Handler` middleware. For other interfaces, use a struct that implements the same interface and holds the wrapped object.

**Example:**

```go
func withTiming(name string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		metrics.Observe(name, time.Since(start))
	})
}
```

### Return `iter.Seq` for iteration [Recommended]

From Go 1.23, iteration over custom collections and paged data returns `iter.Seq` or `iter.Seq2`, and callers use `range` directly. Do not write iterator types with `HasNext()` and `Next()`. When errors need handling, use `iter.Seq2[T, error]`.

**Source:** Go Release Notes, Go Standard Library Docs
