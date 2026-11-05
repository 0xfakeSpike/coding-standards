# Go security

For general security requirements, see [backend security rules](../../backend/index.md). This document covers only Go-specific practices.

## Language-specific risks

### Generate HTML with html/template [Required]

`html/template` escapes automatically by context. `text/template` does not escape. Do not convert user input to `template.HTML` to bypass escaping.

**Source:** Go Standard Library Docs

### Use placeholders in SQL [Required]

Pass parameters through `database/sql` placeholders, not by concatenating with `fmt.Sprintf`. Placeholder syntax depends on the driver, such as `?` or `$1`. Use an allowlist for dynamic table names and sort fields.

**Example:**

Avoid:

```go
query := fmt.Sprintf("SELECT id FROM users WHERE email = '%s'", email)
rows, err := db.QueryContext(ctx, query)
```

Prefer:

```go
rows, err := db.QueryContext(ctx, "SELECT id FROM users WHERE email = ?", email)
```

**Tooling:** `gosec`

### Do not join user paths directly [Required]

On Go 1.24+, use `os.Root` to confine file operations to a directory. It blocks symlinks that point outside. `filepath.IsLocal` only does a lexical check. When extracting archives, check each entry's path.

**Example:**

```go
root, err := os.OpenRoot(uploadDir)
if err != nil {
	return err
}
defer root.Close()
f, err := root.Open(name)
```

**Tooling:** `gosec`'s G304

**Source:** Go Release Notes, Go Standard Library Docs

### Do not disable TLS certificate verification [Required]

Do not set `InsecureSkipVerify: true` in production code. In tests, use `Client()` from `httptest.NewTLSServer`.

**Tooling:** `gosec`'s G402

**Source:** gosec Docs

### Compare secrets with constant-time functions [Required]

For tokens and signatures, use `crypto/subtle.ConstantTimeCompare`, not `==` or `bytes.Equal`.

**Source:** Go Standard Library Docs

### Keep sensitive data out of logs and errors [Required]

Do not write passwords or tokens into error text or logs. Types with sensitive fields implement `slog.LogValuer` or `String` to redact them. Also take care when printing structs with `%+v`.

**Source:** Google Go Style Guide

### Do not expose pprof [Required]

Importing `net/http/pprof` registers it on `http.DefaultServeMux`. Do not use the default mux on business ports. Put debug endpoints on a separate port that listens only on the internal network.

**Source:** Go Standard Library Docs

### Avoid unsafe and cgo unless necessary [Recommended]

They bypass type and memory safety. Do not use them without performance data or a required C library integration.

## Random numbers

### Use crypto/rand for security purposes [Required]

Use `crypto/rand` for keys, tokens, verification codes, and session IDs. Do not use `math/rand` or `math/rand/v2`, even for one-time values. When you need text, use `rand.Text()` (Go 1.24+).

**Tooling:** `gosec`

**Source:** Go Code Review Comments, Google Go Style Guide, Go Standard Library Docs

### Use math/rand/v2 for simulation and jitter [Recommended]

For non-security cases, use `math/rand/v2` (Go 1.22+). The global generator is already randomly seeded, so do not call `Seed` by hand.

**Source:** Go Release Notes

### Use the standard library for UUIDs [Recommended]

On Go 1.27+, use the standard library `uuid` package, whose random part comes from a secure random number generator. On older versions, use a mature third-party library.

**Source:** Go Release Notes

## Serialization

### Limit input size before decoding [Required]

Limit untrusted input with `http.MaxBytesReader` or `io.LimitReader` before decoding, to prevent memory exhaustion.

**Source:** Go Standard Library Docs

### Be explicit about fields when decoding JSON [Recommended]

For strict APIs, use `Decoder.DisallowUnknownFields`. Key matching in `encoding/json` is case-insensitive, so do not rely on it to distinguish fields. On Go 1.27+, `encoding/json/v2` rejects invalid UTF-8 and duplicate keys by default. Consider preferring it when parsing untrusted JSON.

**Source:** Go Standard Library Docs, Go Release Notes

### Exclude secret fields from serialization [Required]

Tag sensitive fields `json:"-"`. For how response structs relate to database entities, see [layering.md](../../backend/architecture/layering.md).

### Do not parse untrusted data with gob [Required]

`encoding/gob` is not hardened against malicious input and is outside the scope of the Go security policy.

**Source:** Go Standard Library Docs

### Write fuzz tests for functions that parse untrusted input [Recommended]

`FuzzXxx` runs its seed corpus under `go test`, and `go test -fuzz=FuzzParse` starts mutation. Keep targets fast and deterministic, and keep crash samples in `testdata/fuzz/`.

**Source:** Go Fuzzing, Go Security Best Practices

## Command execution

### Pass an argument list to exec.Command [Required]

Do not splice user input into `sh -c`. `os/exec` does not go through a shell and passes arguments as they are.

**Example:**

Avoid:

```go
cmd := exec.CommandContext(ctx, "sh", "-c", "convert "+name+" out.png")
```

Prefer:

```go
cmd := exec.CommandContext(ctx, "convert", "--", name, "out.png")
```

**Tooling:** `gosec`'s G204

**Source:** Go Standard Library Docs

### Do not use user input as a program name [Required]

Use a constant or an allowlist for the program name. The target program treats an argument starting with `-` as an option. Separate with `--` when the target supports it. A program resolved through PATH to the current directory returns `exec.ErrDot`. To run it, write `./prog`. Do not disable this protection.

**Source:** Go Standard Library Docs

### Run subprocesses with a context [Required]

**Applies to:** production service

Use `exec.CommandContext` to set a timeout, so subprocesses cannot hang forever.

**Source:** Go Standard Library Docs

## Static scanning

### Run govulncheck in CI [Recommended]

**Applies to:** team project

`govulncheck ./...` uses the call graph and reports only vulnerabilities actually called, so noise is low. Also scan released binaries.

**Tooling:** Register with `go get -tool golang.org/x/vuln/cmd/govulncheck`, then run `go tool govulncheck ./...`.

**Source:** govulncheck Tutorial, Go Security Best Practices

### Run gosec in CI [Recommended]

**Applies to:** team project

Enable it through the `golangci-lint` configuration in [style.md](style.md). Mark false positives with `//nolint:gosec` or `#nosec`, with the reason after the directive.

**Source:** Go Security Best Practices, gosec Docs
