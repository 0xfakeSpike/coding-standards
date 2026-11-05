# Go testing

For race detection requirements, see [concurrency.md](concurrency.md).

## Frameworks

### Prefer the standard testing package [Recommended]

Use `if got != want` with `t.Errorf`, and `github.com/google/go-cmp/cmp` for structural comparison. When a project has standardized on `testify`, stay consistent and do not mix. Do not write a homegrown assertion framework.

**Why:** Assertion libraries tend to produce uninformative failure messages, and they also stop tests early.

**Source:** Google Go Style Guide, Go Code Review Comments

### Make failure messages enough to locate the problem [Recommended]

State the function name, input, actual value, and expected value, with actual first, in the form `Foo(%q) = %v, want %v`. A reader should be able to tell what went wrong without reading the test source.

**Source:** Go Code Review Comments, Google Go Style Guide

### Compare semantics, not unstable output [Recommended]

Compare structs and slices as a whole with `cmp.Diff`, not field by field by hand. Parse JSON before comparing, and do not compare bytes. Check errors with `errors.Is`, and do not compare error text.

**Source:** Google Go Style Guide

## Organization

### Keep test files with the code under test [Recommended]

Tests for `foo.go` go in `foo_test.go`, and test data goes in `testdata/`. Use `package foo` when you need unexported identifiers, and `package foo_test` when testing only the public API or avoiding an import cycle.

**Source:** Google Go Style Guide

### Continue with t.Error, stop with t.Fatal [Recommended]

Use `t.Error` by default so one run reports all problems. Use `t.Fatal` only when later checks would be meaningless. Do not use `panic` in tests.

**Source:** Google Go Style Guide, Uber Go Style Guide

### Do not call t.Fatal from a new goroutine [Required]

`t.FailNow` and `t.Fatal` may be called only from the goroutine running the test. In other goroutines, use `t.Error` and return.

**Tooling:** `go vet`'s `testinggoroutine`

**Source:** Google Go Style Guide

### Call t.Helper in helper functions [Recommended]

Write `t.Helper()` on the first line of test helpers, so failures point to the call site. Put `t` after `ctx` and before other parameters. If environment setup fails, call `t.Fatal` directly.

**Source:** Google Go Style Guide

### Leave cleanup to testing [Recommended]

Use `t.Cleanup`, `t.TempDir`, and `t.Setenv`, and use `t.Context()` (Go 1.24+) as the test's context. This context is canceled before `Cleanup` runs, so use `context.Background()` during cleanup.

**Source:** Google Go Style Guide, Go Standard Library Docs

### Keep only setup every test case needs in TestMain [Recommended]

For test independence requirements, see [testing-strategy.md](../../general/testing-strategy.md). Subtests must be able to run on their own. Do not put setup that only some test cases need in `TestMain` or package-level variables.

**Source:** Google Go Style Guide

## Parameterization

### Use table-driven tests when inputs and outputs vary [Recommended]

Use `t.Run` to name each row, with readable names.

**Example:**

```go
func TestShippingFee(t *testing.T) {
	tests := []struct {
		name        string
		amountCents int64
		wantCents   int64
	}{
		{"below free shipping threshold", 9999, 800},
		{"meets free shipping threshold", 10000, 0},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := ShippingFee(tt.amountCents); got != tt.wantCents {
				t.Errorf("ShippingFee(%d) = %d, want %d", tt.amountCents, got, tt.wantCents)
			}
		})
	}
}
```

**Source:** Google Go Style Guide, Uber Go Style Guide, Go Code Review Comments

### Keep branching logic out of table-driven tests [Recommended]

When the loop body has conditional assertions, or a table field is used by only some test cases, split into multiple tables or test functions.

**Source:** Uber Go Style Guide

## Test doubles

### Prefer real implementations [Recommended]

Do not use a double when you can use the real type. Use `httptest` for HTTP, and real transport to a local test server for RPC.

**Source:** Google Go Style Guide, Gin Docs

### Back doubles with small consumer-side interfaces [Recommended]

The package under test defines an interface with only the methods it needs, and hand-written fakes go in `_test.go`. When several packages share one, put it in a dedicated package named `xxxtest`, such as `creditcardtest`. Do not leave test backdoors on production types.

**Source:** Go Code Review Comments, Google Go Style Guide

### Control time and randomness through injection [Recommended]

Inject with a `now func() time.Time` field. Do not change package-level variables, and do not wait with a real `time.Sleep`. For concurrent and timed logic, use `testing/synctest` (Go 1.25+), which runs in an isolated environment with a virtual clock.

**Example:**

```go
func TestExpiry(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		c := NewCache(time.Minute)
		c.Put("k", 1)
		time.Sleep(2 * time.Minute)
		synctest.Wait()
		if got, ok := c.Get("k"); ok {
			t.Errorf(`Get("k") = %v, true, want expired`, got)
		}
	})
}
```

**Source:** Uber Go Style Guide, Go Release Notes

## Integration tests

### Run integration tests separately from unit tests [Recommended]

Mark them with `//go:build integration`, or start with `if testing.Short() { t.Skip() }`. The default `go test ./...` does not depend on external services.

### Use real dependencies [Recommended]

Use real instances in containers for databases and message queues. SQL in `Store` must have run against a real database. Isolate each test case with a transaction or separate data, and do not depend on execution order.

### Check for goroutine leaks [Recommended]

Packages that start background goroutines check with `goleak` in tests.

**Source:** Uber Go Style Guide

## Benchmarks

### Use b.Loop in benchmarks [Recommended]

On Go 1.24+, write `for b.Loop()` instead of `for range b.N`. Setup before the loop is not timed, and manual `ResetTimer` is not needed. Use `b.ReportAllocs()` to see allocations.

**Example:**

```go
func BenchmarkEncode(b *testing.B) {
	order := sampleOrder()
	b.ReportAllocs()
	for b.Loop() {
		if _, err := Encode(order); err != nil {
			b.Fatal(err)
		}
	}
}
```

**Source:** Go Release Notes, Go Standard Library Docs

### Prove optimizations with data [Recommended]

Attach before-and-after benchmark results to performance changes, and compare multiple runs with `benchstat`. Optimize only hot paths.

**Source:** Uber Go Style Guide, Google Go Style Guide
