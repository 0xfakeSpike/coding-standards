# Python types

This document sets rules for type annotations, None handling, generics, enums, data classes, and boundary validation. Version requirements are stated in the rules. A project's minimum version is set by `requires-python`.

## Using the type system

### Fully annotate public APIs and check them in CI [Recommended]

Write complete parameter and return types for public functions and methods. CI runs `mypy --strict` or pyright's `strict` mode. Strict also checks internal functions. Relax it per module when dependencies lack types. Do not annotate `self` and `cls`.

**Why:** Annotations have no effect at runtime. Without a checker, they are just text that may go stale.

**Tooling:** `mypy --strict`; pyright's `typeCheckingMode = "strict"`.

**Source:** Google Python Style Guide, mypy Docs, pyright Docs

### Use modern syntax [Recommended]

Write `list[str]`, `dict[str, int]`, and `X | Y`, not `List`, `Dict`, `Union`, or `Optional`. Python 3.10+.

**Tooling:** `UP006`, `UP007`, `UP045`.

**Source:** Python typing Docs, Ruff Docs

### Wide parameters, concrete return values [Recommended]

For parameters, use the widest abstract type that meets the need, for example `Iterable[str]`, `Sequence[str]`, or `Mapping[str, int]`. For return values, use concrete types.

**Source:** Google Python Style Guide

### Describe required capabilities with Protocol [Recommended]

When a caller cares only about a few methods, declare them with `typing.Protocol`, defined by the consumer. Use `abc.ABC` only when you need a shared implementation. `@runtime_checkable` checks only whether attributes exist, not signatures or types, so it cannot serve as validation. This capability is not inherited: a subprotocol that needs it must be marked again.

**Why:** Structural typing does not require implementers to inherit, so modules have no import dependency on each other.

**Example:**

```python
from typing import Protocol


class Notifier(Protocol):
    def send(self, recipient: str, body: str) -> None: ...


def notify_owner(notifier: Notifier, owner: str) -> None:
    notifier.send(owner, "Task completed")
```

**Source:** Python typing Docs

### Minimize Any, cast, and type ignores [Recommended]

Use `Any` only for types that truly cannot be expressed. `cast` does no runtime check. Use it only when the checker cannot infer the type. When ignoring a type error, write the specific error code, for example `# type: ignore[no-untyped-call]`, not a bare `# type: ignore`.

**Tooling:** `PGH003`; mypy's `warn_unused_ignores`.

**Source:** mypy Docs, Ruff Docs

### Mark subclass methods with override [Recommended]

Add `@typing.override` when overriding a parent class method. Python 3.12+.

**Why:** When the parent method is renamed, the checker reports the subclass instead of letting it silently stop working.

**Source:** Python typing Docs

### Put annotation-only imports under TYPE_CHECKING [Recommended]

To resolve circular imports or avoid expensive imports, put the imports under `if TYPE_CHECKING:`. When annotations use these names, quote them or add `from __future__ import annotations`. Python 3.14 and later do not need this.

**Exceptions:** Pydantic and FastAPI resolve annotations at runtime, so types used in models and route signatures must be actually imported.

**Tooling:** `TC001`, `TC004`.

**Source:** Python typing Docs, Ruff Docs

## None handling

### Write nullable types explicitly and narrow first [Recommended]

When the default is `None`, write `x: int | None = None`, not `x: int = None`. PEP 484 no longer recommends implicit nullable types. Narrow with `is None` before use. Do not bypass the checker with `assert` or `cast`.

**Tooling:** `RUF013`.

**Source:** PEP 484, Ruff Docs

## Generics

### Use type parameter syntax [Recommended]

On Python 3.12+, use `def first[T](...)`, `class Box[T]`, and `type Pairs[T] = ...`. When you need to support 3.11 and below, use `TypeVar`, and start internal type variables with an underscore. A type variable that appears only once in a signature is pointless. Write the concrete type or `object` directly.

**Example:**

```python
from collections.abc import Callable, Sequence


def first[T](items: Sequence[T]) -> T:
    return items[0]


def logged[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    return func


type UserById[T] = dict[int, T]
```

**Tooling:** `UP040`, `UP046`, `UP047`.

**Source:** Python typing Docs, Ruff Docs

## Enums

### Use enums for fixed values [Recommended]

Express status and type values with `enum.Enum`. Use `StrEnum` (Python 3.11+) when they need to serialize to strings. When values are constrained only at the type level, you may use `Literal`. When branching on an enum or union type, call `typing.assert_never` (Python 3.11+) in the last branch, so the checker flags omissions when members are added.

**Source:** Python Standard Library Docs, Python typing Docs

## Data classes

### Do not pass bare dicts for structured data [Recommended]

Use `dataclass` for internal data and Pydantic for external input. Use `TypedDict` only for data that really is dict-shaped, such as JSON fragments. For data that does not need to change, write `frozen=True, slots=True`. Add `kw_only=True` when there are many fields. The last two parameters require Python 3.10+.

**Why:** In a bare `dict`, neither field spelling nor types are checked.

**Example:**

```python
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True, kw_only=True)
class Report:
    title: str
    tags: list[str] = field(default_factory=list)
```

**Tooling:** `RUF008`, `RUF009`.

**Source:** Python Standard Library Docs

## Type boundary validation

### Convert external data to validated types at the boundary [Required]

When HTTP requests, files, messages, and environment variables enter the system, parse them immediately into Pydantic models or dataclasses. Internal code accepts only validated types and does not validate again.

**Why:** Type annotations do no runtime checking, and external data does not become trustworthy because it has an annotation.

**Example:**

```python
from pydantic import BaseModel


class SignupForm(BaseModel):
    email: str
    age: int


def parse_signup(payload: bytes) -> SignupForm:
    return SignupForm.model_validate_json(payload)
```

**Source:** Pydantic Docs

### Distinguish same-typed identifiers with NewType [Recommended]

Distinguish values of the same type but different meaning with `typing.NewType`, for example `UserId` and `OrderId`. It has no runtime overhead and does no validation.

**Source:** Python typing Docs
