# Go concurrency

## Concurrency model

### Prefer synchronous functions [Recommended]

Have functions return results directly and let the caller decide on concurrency. A caller can easily add a goroutine, but removing unneeded concurrency is hard.

**Source:** Go Code Review Comments, Google Go Style Guide

### Give every goroutine an exit path and a waiter [Required]

Decide at launch when it ends and who waits for it. Garbage collection does not end a blocked goroutine, and a leaked goroutine keeps holding memory and resources.

**Example:**

Avoid:

```go
go func() {
	for {
		flush()
		time.Sleep(delay)
	}
}()
```

Prefer:

```go
func (w *Worker) Run(ctx context.Context) error {
	ticker := time.NewTicker(w.delay)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			w.flush(ctx)
		case <-ctx.Done():
			return ctx.Err()
		}
	}
}
```

**Tooling:** Use `go.uber.org/goleak` in tests.

**Source:** Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide, Go Release Notes

### Wait for goroutines you started before returning [Recommended]

A goroutine does not outlive the function that started it. For multiple tasks, use `sync.WaitGroup`. On Go 1.25+, use `wg.Go(f)` instead of paired `Add` and `Done`. When a goroutine must run long term, have one object own it and provide `Close` or `Shutdown` to signal exit and wait.

**Source:** Google Go Style Guide, Uber Go Style Guide, Go Standard Library Docs

### Use errgroup for a group of tasks that can fail [Recommended]

`errgroup.WithContext` cancels the remaining tasks on the first error, and `Wait` returns the first error. The derived ctx is invalid after `Wait` returns. Do not carry it into later code.

**Example:**

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(maxFetches)
for _, addr := range addrs {
	g.Go(func() error {
		return fetch(ctx, addr)
	})
}
return g.Wait()
```

**Source:** errgroup Docs, Google Go Style Guide

### Use a mutex for simple shared state [Recommended]

Channels suit transferring data ownership and signaling. For state such as counters and caches, a lock is more direct. Do not force a channel.

**Source:** Effective Go

### Let the sender close the channel [Required]

Sending on a closed channel panics. With multiple senders, have one coordinator close it after all sends finish. Write `<-chan T` or `chan<- T` in function parameters so the compiler prevents misuse.

**Source:** Go Blog: Pipelines, Google Go Style Guide

## Cancellation and timeouts

### Pass ctx as the first parameter, not in a struct [Recommended]

Write function signatures as `func F(ctx context.Context, ...)`. Do not use a custom Context type, do not pass nil, and use `context.TODO()` when unsure. Use `r.Context()` in HTTP handlers and `t.Context()` (Go 1.24+) in tests.

**Why:** A ctx in a struct hides its lifetime, and callers cannot set a deadline or cancellation per call.

**Exceptions:** When a method signature must match a standard library or third-party interface.

**Tooling:** `contextcheck`

**Source:** Go Standard Library Docs, Go Code Review Comments, Google Go Style Guide, Go Blog: Contexts and Structs

### Create the root context only at entry points [Recommended]

Use `context.Background()` only in entry functions such as `main` and in `init`. Middle layers use the ctx passed by the caller. For work that continues after the request ends, use `context.WithoutCancel(ctx)` (Go 1.21+), which keeps values and breaks cancellation.

**Source:** Google Go Style Guide, Go Standard Library Docs

### Defer cancel right after deriving a context [Required]

Write `defer cancel()` on the line after `ctx, cancel := context.WithTimeout(ctx, d)`. Otherwise resources are not released until the parent context ends.

**Tooling:** `go vet`'s `lostcancel`

**Source:** Go Standard Library Docs

### Use context.Value only for request-scoped data [Recommended]

Store only data that crosses processes and APIs, such as traceId and authentication info. Do not pass function dependencies or optional parameters. Use an unexported custom type for the key, such as `type traceIDKey struct{}`, not a string.

**Source:** Go Standard Library Docs, Go Code Review Comments

### Make every blocking operation respond to cancellation [Required]

**Applies to:** production service

Put channel sends and receives in a `select` that also listens on `ctx.Done()`. Check `ctx.Err()` periodically in long loops. Use ctx-aware APIs for I/O, such as `QueryContext` and `http.NewRequestWithContext`.

**Source:** Go Blog: Pipelines, Go Blog: Context

## Shared state

### Synchronize concurrent access [Required]

One goroutine writing while another reads or writes the same memory is a data race. This is a defect, not a performance trade-off. Serialize access with channels, `sync`, or `sync/atomic`. For atomics, use `atomic.Int64` and `atomic.Bool` (Go 1.19+), not `atomic.AddInt64` on bare integers.

**Source:** Go Memory Model

### Lock maps for concurrent reads and writes [Required]

Concurrent reads and writes on a plain `map` can crash the runtime outright. `sync.Map` suits only write-once, read-many use or disjoint key sets. Otherwise use `map` with `sync.RWMutex` or `sync.Mutex`.

**Source:** Go Standard Library Docs

### Do not copy structs containing sync.Mutex [Required]

Use pointer receivers and pass by pointer for types that contain a lock. `sync.WaitGroup` and `atomic.Int64` must not be copied either.

**Tooling:** `go vet`'s `copylocks`

**Source:** Go Standard Library Docs, Google Go Style Guide

## Resource limits

### Cap concurrency [Required]

**Applies to:** production service

Do not start goroutines without limit, one per input item. Use `errgroup.SetLimit`, a fixed number of workers, or a buffered semaphore.

**Source:** errgroup Docs

### Default channel buffers to 0 or 1 [Recommended]

For a larger buffer, state the basis for its size and who blocks when it fills. A buffer does not fix a leak. It only delays it.

**Source:** Uber Go Style Guide, Go Blog: Pipelines

### Do not set GOMAXPROCS by hand in containers [Recommended]

On Linux, Go 1.25+ sets it from the cgroup CPU limit by default and updates it when the limit changes. Setting it by hand turns this behavior off.

**Source:** Go Release Notes

## Common races

### Run go test -race in CI [Recommended]

**Applies to:** team project

The detector finds only paths that actually run, so pair it with enough tests and concurrent scenarios. Memory use grows about 5 to 10 times and run time about 2 to 20 times, so it is usually not used for production builds.

**Tooling:** `go test -race ./...`

**Source:** Go Race Detector
