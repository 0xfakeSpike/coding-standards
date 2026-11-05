# Python idioms

This document covers language usage and common pitfalls. For design-level principles, see [code-design.md](../../general/code-design.md).

## Idioms

### Do not write functions that only forward a call [Required]

When a function body only passes its arguments unchanged to another function, call the target function directly. Do not add this wrapper.

**Example:**

Avoid:

```python
def get_user(user_id: int) -> User:
    return repository.find_user(user_id)
```

Prefer:

```python
user = repository.find_user(user_id)
```

**Source:** Team

### Do not write a class when a function will do [Recommended]

Write stateless logic as module-level functions. Do not write classes with only static methods.

### Use with to manage resources [Required]

Manage files, locks, and connections with `with` or `async with`. Do not rely on destructors to release them.

**Tooling:** `SIM115`.

**Source:** PEP 8, Google Python Style Guide

### Use is for None checks [Recommended]

Write `x is None` and `x is not None`, not `x == None`. Check for empty sequences with `if not items:`. Parameters that default to `None` must be checked with `is None`, because `0`, `""`, and `[]` are also falsy. Do not write `x = x or []`.

**Tooling:** `E711`.

**Source:** PEP 8, Google Python Style Guide

### Use keyword-only arguments for easily confused parameters [Recommended]

Put boolean flags and parameters with unclear meaning after `*`, for example `def create_user(data: UserCreate, *, send_email: bool = False) -> User: ...`.

**Tooling:** `FBT001`, `FBT002`.

## Collections

### Use built-in tools and the right containers [Recommended]

Use comprehensions only for one loop with a simple condition. With multiple `for` clauses or multiple conditions, rewrite as a plain loop. Use `enumerate` for indexes and `any` and `all` for checks. Write `strict=True` on `zip` when the inputs should have equal length (Python 3.10+). Use `set` for membership checks, `Counter` for counting, `defaultdict` for grouping, and `deque` for queues.

**Tooling:** `B905`.

**Source:** Google Python Style Guide

## Strings

### Use modern forms for strings and paths [Recommended]

Format with f-strings, except in logging. See [errors.md](errors.md). For concatenation in a loop, use `"".join(parts)`. For prefixes and suffixes, use `startswith` and `removeprefix`, not slice comparison. Use `pathlib.Path` for paths, not string concatenation. When reading and writing text, pass `encoding="utf-8"` explicitly, because the default may depend on the system locale.

**Tooling:** `PTH` rule group, `PLW1514`.

**Source:** PEP 8, Python Standard Library Docs

## Dates and times

### Use timezone-aware datetimes [Required]

Use `datetime.now(UTC)` (`datetime.UTC` requires Python 3.11+), not `datetime.utcnow()`, which returns a naive value. For named time zones, use `zoneinfo.ZoneInfo("Asia/Shanghai")`, not hand-written fixed offsets. In environments without a system time zone database (such as Windows), declare a `tzdata` dependency. Measure elapsed time with `time.monotonic()`, because `time.time()` is affected by system clock adjustments.

**Why:** Many `datetime` methods treat naive values as local time, so results change on a different machine.

**Tooling:** `DTZ` rule group.

**Source:** Python Standard Library Docs

## Numbers and money

### Do not use floats for money [Required]

Use `decimal.Decimal` or an integer count of the smallest currency unit. Construct `Decimal` from strings, and set precision and rounding explicitly with `quantize`.

**Why:** `Decimal(0.1)` carries the binary floating-point error over as-is.

**Example:**

Avoid:

```python
total = Decimal(19.99) * 3
```

Prefer:

```python
total = (Decimal("19.99") * 3).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

**Source:** Python Standard Library Docs

## Common pitfalls

### Do not use assert for runtime validation [Required]

`python -O` removes `assert`. To validate parameters and business conditions, use `if` and raise an exception. `assert` in test code is exempt.

**Tooling:** `S101`.

**Source:** Google Python Style Guide
