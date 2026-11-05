# Go types

## Using the type system

### Define interfaces on the consumer side [Recommended]

Put an interface in the package that calls it. List only the methods actually used, keep it small, and leave it unexported when used only inside the package. Functions accept interfaces and return concrete types, so adding methods does not affect consumers.

**Example:**

Avoid:

```go
package order

type Store interface {
	ByID(ctx context.Context, id int64) (Order, error)
	Save(ctx context.Context, o Order) error
	Delete(ctx context.Context, id int64) error
}

func NewStore(db DBTX) Store {
	return &sqlStore{db: db}
}
```

Prefer:

```go
package order

type Store struct {
	db DBTX
}

func NewStore(db DBTX) *Store {
	return &Store{db: db}
}

package invoice

type orderFinder interface {
	ByID(ctx context.Context, id int64) (order.Order, error)
}
```

**Exceptions:** The provider defines the interface when the interface itself is the product, such as `io.Writer`. Factory functions that decide the concrete type at runtime, and `error`, also return interfaces.

**Source:** Go Code Review Comments, Google Go Style Guide

### Do not define interfaces early [Recommended]

Extract an interface only with two or more implementations, or when a consumer truly needs to swap one. Concepts such as "service" and "storage" do not call for an interface of the same name. Do not define an interface on the implementation side for mocking, and do not wrap a generated RPC client in another hand-written interface.

**Why:** Without real usage, you cannot tell whether an interface is necessary, and you cannot pin down its method set.

**Source:** Go Code Review Comments, Google Go Style Guide

### Check at compile time when a type must implement an interface [Recommended]

When an exported type promises to implement an interface, write `var _ http.Handler = (*Handler)(nil)`.

**Source:** Effective Go, Uber Go Style Guide

### Do not write thin shell types or forwarding functions [Required]

Delete structs that hold one field and forward to it, and functions whose body only forwards once. Callers use the original object directly.

**Example:**

Avoid:

```go
type UserService struct {
	store *Store
}

func (s *UserService) ByID(ctx context.Context, id int64) (User, error) {
	return s.store.ByID(ctx, id)
}
```

Prefer:

```go
func NewHandler(users *user.Store) *Handler {
	return &Handler{users: users}
}
```

**Source:** Team

### Keep receiver types consistent [Recommended]

Use a pointer receiver when you need to modify the receiver, when the type has non-copyable fields such as `sync.Mutex`, or when the struct is large. Use a value receiver for `map`, functions, `chan`, and small immutable value types. When unsure, use a pointer. Do not mix receiver kinds on one type. Do not pass pointers to save a few bytes: change parameters such as `*string` and `*io.Reader` to values. Do not copy a `T` whose methods are on `*T`, such as `bytes.Buffer`.

**Source:** Go Code Review Comments, Google Go Style Guide

### Embed with care [Recommended]

Do not embed implementation-detail types in public structs, and do not embed mutexes in any struct. Embed only when all the embedded type's exported methods belong on the outer type. Otherwise use a named field.

**Why:** Embedding makes the inner type's methods and fields part of the outer API.

**Source:** Uber Go Style Guide

## Empty values

### Make the zero value usable [Recommended]

Design types so `var x T` works directly. `sync.Mutex` and `bytes.Buffer` are models. Use a value field for a mutex, not `*sync.Mutex`. When a zero value cannot work, provide a constructor.

**Source:** Effective Go, Uber Go Style Guide, Google Go Style Guide

### Use nil slices for empty and check with len [Recommended]

Writing to a nil map panics, but reading is safe. Write local empty slices as `var t []string` and check emptiness with `len(s) == 0`. APIs do not make callers distinguish nil from empty slices.

**Exceptions:** When JSON needs to output `[]`, initialize to `[]string{}`. A nil slice serializes to `null`.

**Source:** Go Code Review Comments, Google Go Style Guide, Uber Go Style Guide

### Do not return a nil pointer as a nil interface [Required]

A nil pointer of a concrete type stored in an interface does not equal nil. This is especially dangerous when a function returns `error`. With no error, write `return nil` directly, and declare the return type as `error`.

**Source:** Go FAQ, Google Go Style Guide

### Express missing values with an extra return value [Recommended]

Use `(value, ok)` or `error` to indicate no result, not in-band signals such as `-1`, `""`, or `nil`.

**Source:** Go Code Review Comments, Google Go Style Guide

## Generics

### Use generics only with several real types [Recommended]

With only one type in use, write concrete code first. When several types share behavior, consider interfaces first. Use `any` in new code, not `interface{}`. Use generics to replace `any` plus type assertions. Do not use them to build error-handling frameworks or assertion libraries.

**Source:** Google Go Style Guide, Go Release Notes

## Enums

### Use a named type with iota, starting at 1 [Recommended]

Reserve the zero value for "unset", so a missed assignment is not taken as a valid state.

**Example:**

```go
type Status int

const (
	StatusPending Status = iota + 1
	StatusPaid
	StatusRefunded
)
```

**Exceptions:** Start at 0 when the zero value is itself a sensible default.

**Tooling:** `exhaustive` checks that `switch` branches are complete, and `stringer` generates `String` methods.

**Source:** Uber Go Style Guide

## Data classes

### Write field names in struct literals [Recommended]

Types from other packages must use field names, and zero-value fields are omitted. Write a zero-value struct as `var user User`.

**Tooling:** `go vet`'s `composites` check.

**Source:** Uber Go Style Guide, Google Go Style Guide

### Tag serialized fields [Recommended]

Write tags on all fields involved in JSON or YAML encoding, so renaming a Go field does not affect the external format. When zero values should be omitted, Go 1.24+ can use `omitzero`, which handles `time.Time` correctly.

**Source:** Uber Go Style Guide, Go Release Notes

### Use a config struct or functional options for optional parameters [Recommended]

Default to a config struct. Consider functional options only when most callers need no options, since they take more code. Do not put `context.Context` in a config struct.

**Source:** Google Go Style Guide, Uber Go Style Guide

## Type boundary validation

### Parse into domain types at the entry point [Required]

Parse and validate external input once at the boundary and convert it to domain types. Do not repeat the checks internally. Constructors return `error` on failure.

### Use the comma-ok form for type assertions [Required]

A single-value assertion panics on a type mismatch. Write `v, ok := x.(T)`.

**Source:** Uber Go Style Guide
