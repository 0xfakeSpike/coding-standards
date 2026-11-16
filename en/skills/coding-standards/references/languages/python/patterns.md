# Python design patterns

This file covers the idiomatic forms of design patterns in Python. For when to use which pattern, see [design-patterns.md](../../general/design-patterns.md). Functions, modules and generators are first-class objects, so most patterns need no class, see "Do not write a class when a function will do" in [idioms.md](idioms.md). Use `Protocol` for interfaces, see "Describe required capabilities with Protocol" in [types.md](types.md).

## Structure

### Use a module for singletons, not `__new__` or a metaclass [Recommended]

A module is imported once, so a module-level object exists only once. When it must be created lazily, decorate a function with no parameters with `functools.cache`. Do not keep mutable business state at module level, see "Avoid mutable global state" in [style.md](style.md).

**Example:**

```python
import functools


@functools.cache
def get_settings() -> Settings:
    return Settings.from_env()
```

**Source:** Python Standard Library Docs

### Use `classmethod` for alternative constructors and keyword arguments for options [Recommended]

Do not write `XxxFactory` classes or builders. To build an object from another format, write a `classmethod` such as `from_dict` or `from_row`. For optional parameters, use keyword-only arguments with defaults or a `dataclass`, see "Use keyword-only arguments for easily confused parameters" in [idioms.md](idioms.md). To change a few fields of an immutable object, use `dataclasses.replace`.

**Example:**

```python
from dataclasses import dataclass
from typing import Self


@dataclass(frozen=True, slots=True)
class Money:
    cents: int
    currency: str

    @classmethod
    def from_row(cls, row: Row) -> Self:
        return cls(cents=row.amount_cents, currency=row.currency)
```

### Access attributes directly, and use `property` only when computing [Recommended]

Do not write `get_name()` and `set_name()`. Access fields directly. When you need to compute, validate or keep an invariant, turn the field into a `@property`, and callers do not change.

**Source:** Google Python Style Guide

## Behavior

### Use functions or a dict of functions for strategies [Recommended]

Write one function per case and put them in a dict keyed by an enum. A dict has no compile-time completeness check, so add a test that every enum member has a function. When a strategy needs several methods, describe it with a `Protocol` and implement it in separate classes.

**Example:**

```python
from collections.abc import Callable

EXPORTERS: dict[ExportFormat, Callable[[list[Row]], bytes]] = {
    ExportFormat.CSV: to_csv,
    ExportFormat.XLSX: to_xlsx,
}


def test_every_format_has_exporter() -> None:
    assert set(EXPORTERS) == set(ExportFormat)
```

### Use an enum and `match` for state machines [Recommended]

Express states with `enum.Enum`. The transition function branches on the state with `match`, each branch returns the next state, and a final `assert_never` lets the type checker find unhandled states. When states need different data, use a union of several `dataclass`es instead of an enum and match on class patterns.

**Example:**

```python
from enum import Enum, auto
from typing import assert_never


class Step(Enum):
    AWAITING_TOKEN = auto()
    UPLOADING = auto()
    MERGING = auto()
    DONE = auto()


def advance(step: Step, job: UploadJob) -> Step:
    match step:
        case Step.AWAITING_TOKEN:
            job.token = fetch_token()
            return Step.UPLOADING
        case Step.UPLOADING:
            job.parts = upload_parts(job.token)
            return Step.MERGING
        case Step.MERGING:
            job.file_id = merge_parts(job.token, job.parts)
            return Step.DONE
        case Step.DONE:
            return step
        case _:
            assert_never(step)
```

**Source:** Python typing Docs

### Use decorators for cross-cutting logic and context managers for paired setup and cleanup [Recommended]

Write timing, retries and caching as function decorators, and keep the original function's metadata with `functools.wraps`. For the "set up, run, clean up" kind of template method, write a `contextlib.contextmanager`. Do not write an abstract base class for subclasses to override steps.

**Example:**

```python
import functools
import time
from collections.abc import Callable


def timed[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            metrics.observe(fn.__qualname__, time.perf_counter() - start)

    return wrapper
```

**Source:** Python Standard Library Docs

### Use generators for iteration [Recommended]

Write iteration over paged data and custom collections as generators with `yield`, and combine them with `itertools`. Do not write iterator classes that implement `__next__` unless the object itself must keep complex iteration state.

**Example:**

```python
from collections.abc import Iterator


def all_orders(client: Client) -> Iterator[Order]:
    cursor: str | None = None
    while True:
        page = client.list_orders(cursor)
        yield from page.items
        if page.next_cursor is None:
            return
        cursor = page.next_cursor
```

**Source:** Python Standard Library Docs
