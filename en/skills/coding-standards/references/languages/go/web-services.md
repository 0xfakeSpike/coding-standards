# Go web services

**Applies to:** production service (all rules in this document)

For API design, see [backend/api/rest.md](../../backend/api/rest.md). This document covers only Go implementation requirements.

## net/http

### Configure http.Server explicitly [Required]

Do not use `http.ListenAndServe`. Its timeouts are all zero values, which means no limit. Set at least `ReadHeaderTimeout`, and set `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` to suit the endpoints. For streaming responses, adjust deadlines individually with `http.ResponseController`.

**Example:**

```go
srv := &http.Server{
	Addr:              cfg.Addr,
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       30 * time.Second,
	WriteTimeout:      30 * time.Second,
	IdleTimeout:       2 * time.Minute,
}
```

**Source:** Go Standard Library Docs

### Use your own ServeMux [Recommended]

Create `http.NewServeMux()`, and do not use `http.DefaultServeMux`, because any imported package can register routes on it. On Go 1.22+, write methods and path parameters directly. Simple services need no third-party router.

**Example:**

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /orders/{id}", h.getOrder)
mux.HandleFunc("POST /orders", h.createOrder)
```

**Source:** Go Release Notes, Go Standard Library Docs

### Keep handlers to protocol conversion [Recommended]

Parse the request, call business logic, and write the response. Business logic does not depend on `http.Request` or `gin.Context`. Pass `r.Context()` to downstream calls.

### Limit request body size [Required]

Limit with `http.MaxBytesReader` or `http.MaxBytesHandler`. Reads past the limit return `*http.MaxBytesError`.

**Source:** Go Standard Library Docs

### Do not expose internals when mapping errors [Required]

Map package errors to status codes. For all unknown errors, return 500 with a generic message, and put details in logs.

## Middleware

### Use the standard middleware signature [Recommended]

Write `func(http.Handler) http.Handler`, and state the order once where middleware is assembled. Pass request-scoped data through `context`. For key requirements, see [concurrency.md](concurrency.md).

### Implement Unwrap when wrapping ResponseWriter [Recommended]

The wrapper type provides `Unwrap() http.ResponseWriter`. Otherwise `http.ResponseController` cannot use underlying capabilities such as `Flush` and `SetWriteDeadline`.

**Source:** Go Standard Library Docs

### Recover panics in the outermost layer [Recommended]

`net/http` recovers handler panics and drops the connection. When you need to return 500 or report the panic, `recover` in the outermost middleware, and re-panic `http.ErrAbortHandler`.

**Source:** Go Standard Library Docs

## Gin

### Assemble middleware explicitly with gin.New [Recommended]

`gin.Default()` comes with Logger and Recovery. When you need structured logs, use `gin.New()`, then `Use` your own logger and `gin.Recovery()`. Set `GIN_MODE=release` in production. Put Gin in an `http.Server` too, setting timeouts with `Handler: router`, and do not use `router.Run`.

**Source:** Gin Docs

### Bind with the ShouldBind family [Recommended]

`ShouldBindJSON` hands errors back to your code for uniform handling. Methods such as `BindJSON` write a 400 and a `text/plain` response themselves. Put validation rules in `binding` tags, and convert errors to the API's uniform format.

**Source:** Gin Docs

### Pass c.Request.Context() downstream [Required]

By default, `Done` on `gin.Context` returns nil and does not propagate cancellation. Pass `c.Request.Context()` to the business layer. The business layer does not accept `*gin.Context`.

**Source:** Gin Docs

### Use c.Copy() in goroutines [Required]

`gin.Context` is reused. To use it outside the handler, call `c.Copy()` first, and only read from the copy.

**Source:** Gin Docs

### Do not trust client-provided information [Required]

Gin trusts all proxies by default, and `ClientIP()` can be spoofed with request headers. List proxy addresses with `SetTrustedProxies`, and pass `nil` when there is no proxy. Do not use an uploaded `file.Filename` as the save path. Generate file names on the server.

**Source:** Gin Docs

## gRPC services

### Use grpc.NewClient on the client [Recommended]

On grpc-go 1.63+, use `grpc.NewClient`, not `grpc.Dial` or `WithBlock`. `ClientConn` is a virtual connection that reconnects automatically, so you need not confirm readiness first. Rely on the RPC result for errors.

**Source:** grpc-go Docs

### Set a deadline on every call [Required]

gRPC has no default deadline, so clients may wait forever. Set one with `context.WithTimeout`. When the server calls downstream, pass the inbound ctx. Long tasks check `ctx.Err()` periodically.

**Source:** gRPC Docs

### Return status errors from the server [Recommended]

In RPC methods or interceptors, map package errors to `status.Error(codes.NotFound, "order not found")`. Plain errors reach the client as `Unknown`. Clients check with `status.Code(err)`, which recognizes wrapped status errors. Business packages do not reference `codes`.

**Source:** gRPC Docs, grpc-go Docs

### Handle cross-cutting logic with interceptors [Recommended]

Assemble logging, authentication, and metrics with `grpc.ChainUnaryInterceptor` and its streaming counterpart.

**Source:** grpc-go Docs

### Add a recovery interceptor to gRPC services [Required]

gRPC does not recover handler panics. One panic takes the process down.

**Source:** grpc-go Docs

## Graceful shutdown

### Listen for signals with signal.NotifyContext [Required]

Shutdown needs a new ctx, because the signal ctx is already canceled at that point. `ListenAndServe` returns `http.ErrServerClosed` as soon as `Shutdown` is called, so the process must wait for `Shutdown` to return before exiting.

**Example:**

```go
func run(ctx context.Context, srv *http.Server) error {
	ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
	defer stop()

	serveErr := make(chan error, 1)
	go func() {
		serveErr <- srv.ListenAndServe()
	}()

	select {
	case err := <-serveErr:
		return err
	case <-ctx.Done():
	}
	stop()

	shutdownCtx, cancel := context.WithTimeout(context.Background(), shutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown http server: %w", err)
	}
	if err := <-serveErr; !errors.Is(err, http.ErrServerClosed) {
		return err
	}
	return nil
}
```

**Source:** Go Standard Library Docs, Gin Docs

### Handle long-lived connections separately [Recommended]

`Shutdown` neither closes nor waits for hijacked connections such as WebSocket. Notify them with `RegisterOnShutdown`.

**Source:** Go Standard Library Docs

### Give gRPC graceful shutdown a fallback [Required]

`GracefulStop` blocks until all RPCs finish. Use a timer to call `Stop` after a timeout. When `health/v1` is registered, call the health service's `Shutdown` before stopping, to set the status to `NOT_SERVING`.

**Example:**

```go
healthSrv.Shutdown()
stopped := make(chan struct{})
go func() {
	grpcServer.GracefulStop()
	close(stopped)
}()
select {
case <-stopped:
case <-time.After(shutdownTimeout):
	grpcServer.Stop()
	<-stopped
}
```

**Source:** gRPC Docs, grpc-go Docs

### Shut down in reverse dependency order [Recommended]

Stop accepting requests first, then wait for background workers to exit, and finally close database and messaging connections. Total time must stay under the platform's forced termination grace period.

## Configuration

### Load and validate once at startup [Recommended]

In `main`, parse environment variables into a config struct, exit immediately if validation fails, and pass it explicitly to components. Business code does not call `os.Getenv`, and `init` does not read configuration. Parse durations with `time.ParseDuration`. Redact configuration when printing it at startup.

**Source:** Uber Go Style Guide, Google Go Style Guide

### Separate environment-specific configuration from code [Recommended]

Values that vary by deployment, such as addresses and credentials, come from environment variables or injected configuration, not code constants.

**Source:** Twelve-Factor App
