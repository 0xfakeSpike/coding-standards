# Go idioms

## Idioms

### Prefer standard library idioms [Recommended]

Do not add a library when the core language is enough. On Go 1.21+, use the built-in `min`, `max`, and `clear`, plus `slices`, `maps`, and `cmp`. Go 1.22+ allows `for i := range n`. For set membership, `map[string]bool` is usually enough.

**Tooling:** `go fix` (Go 1.26+)

**Source:** Google Go Style Guide, Go Release Notes

### Release resources with defer [Required]

Write `defer` right after acquiring a resource, so every return path releases it. The overhead of `defer` is not a reason to avoid it.

**Source:** Uber Go Style Guide, Effective Go

### Do not defer per-iteration resources inside a loop body [Recommended]

`defer` does not run until the function ends, so resources keep piling up. Extract the loop body into a function.

**Example:**

Avoid:

```go
for _, name := range names {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close()
	if err := scan(f); err != nil {
		return err
	}
}
```

Prefer:

```go
func scanFile(name string) error {
	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close()
	return scan(f)
}
```

### Narrow variable scope [Recommended]

For a result used only inside `if`, write `if err := f(); err != nil`. Use `:=` for local assignment and `var` to express a zero value. Use named returns only when results of the same type need telling apart or `defer` must modify them. Use bare `return` only in very short functions.

**Source:** Uber Go Style Guide, Google Go Style Guide, Go Code Review Comments

### Do not use mutable package-level variables [Recommended]

Pass dependencies through constructors or parameters. The clock is no different: use a `now func() time.Time` field instead of replacing a global. Libraries do not make consumers depend on global state.

**Why:** Global state makes tests affect each other and prevents running multiple instances in one process.

**Source:** Uber Go Style Guide, Google Go Style Guide

## Collections

### Slices share their backing array [Required]

A subslice and its original see each other's changes, and `append` overwrites later elements when capacity allows. For isolation, use `slices.Clone` or `copy`. To keep appending, use the full slice expression `s[:n:n]`. Functions such as `slices.Delete` modify in place, so use their return value. Also copy when keeping a small piece of a large buffer long term, or the whole array cannot be collected.

**Example:**

Avoid:

```go
head := items[:2]
head = append(head, extra)
```

Prefer:

```go
head := slices.Clone(items[:2])
head = append(head, extra)
```

**Source:** Effective Go

### Copy slices and maps at boundaries [Recommended]

Copy when storing an argument or returning an internal field. Otherwise outside changes bypass your invariants and locks.

**Example:**

```go
func (s *Stats) Snapshot() map[string]int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return maps.Clone(s.countByName)
}
```

**Source:** Uber Go Style Guide

### Preallocate only when the size is known [Recommended]

When the size is known, write `make([]T, 0, n)` or `make(map[K]V, n)`. Most code does not need this, and over-allocating wastes memory. In performance-sensitive code, decide with benchmarks.

**Source:** Uber Go Style Guide, Google Go Style Guide

## Strings

### Distinguish bytes from characters [Required]

`len(s)` is the byte count, `s[i]` is a byte, and only `for range s` iterates by rune. Truncate strings containing multi-byte text, such as Chinese, by rune.

**Source:** Effective Go

### Declare format strings as constants [Recommended]

Declare format strings kept outside the literal call as `const`, and end custom Printf-style function names in `f`. Otherwise `go vet` cannot check them.

**Tooling:** `go vet`

**Source:** Uber Go Style Guide

## Date and time

### Use time.Time for instants and time.Duration for durations [Recommended]

Do not use bare integers with a unit suffix. When an external format does not support these types, put the unit in the field name, such as `IntervalMillis`. Use RFC 3339 for timestamp strings.

**Source:** Uber Go Style Guide

### Compare with Equal, not == [Required]

`==` compares both the time zone and the monotonic clock reading, so the same instant can compare unequal. For order, use `Before` and `After`.

**Source:** Go Standard Library Docs

### Distinguish calendar addition from duration addition [Required]

For the next calendar day, use `AddDate(0, 0, 1)`. For exactly 24 hours later, use `Add(24 * time.Hour)`. The two differ on a daylight saving transition day.

**Source:** Uber Go Style Guide

## Numbers and money

### Do not use floats for money [Required]

Use integers for the smallest currency unit, such as `int64` cents, or use a decimal library. Put the unit in the type or field name.

**Why:** Binary floating point cannot represent most decimal fractions exactly, and errors appear after summing.

### Watch for integer overflow and truncation [Required]

Go integer overflow raises no error and wraps around. Converting `int64` to `int32` truncates silently. Check the range before converting external numbers.

**Tooling:** `gosec`

## Common pitfalls

### Set timeouts on HTTP clients and close response bodies [Required]

`http.DefaultClient` has a zero `Timeout`, which means no timeout. Create a `Client` once and reuse it. Write `defer resp.Body.Close()` right after checking `err`.

**Tooling:** `bodyclose`, `noctx`

**Source:** Go Standard Library Docs

### Check the Close error when writing files [Required]

A buffered write failure may surface only at `Close`. A plain `defer f.Close()` loses it.
