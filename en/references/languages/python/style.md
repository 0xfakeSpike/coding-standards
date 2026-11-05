# Python style

This document sets rules for tool configuration, formatting, naming, imports, and file organization. For general naming principles, see [naming.md](../../general/naming.md).

## Tool configuration

### Use Ruff for formatting and static checks [Recommended]

**Applies to:** team project

Format with `ruff format`. Run static checks and import sorting with `ruff check`. Use mypy or pyright separately for type checking. Keep the configuration for these tools in `pyproject.toml`, not scattered across multiple files.

**Why:** With scattered configuration, local runs and CI easily read different rules.

**Exceptions:** Projects already using Black, Flake8, and isort may keep them, but keep only one tool per responsibility.

**Source:** Ruff Docs

### Select lint rules explicitly [Recommended]

List rule groups in `lint.select`. Do not use `ALL`. Start from the baseline below and add or remove rules as needed. When using `ruff format`, do not enable rules that conflict with it: `W191`, `E111`, `E114`, `E117`, `Q`, `COM812`, `COM819`.

**Why:** `ALL` silently enables new rules when Ruff is upgraded, so the same code may suddenly fail the gate.

**Example:**

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = [
    "E", "F", "I", "N", "UP", "B", "A", "SIM", "RUF",
    "ASYNC", "BLE", "DTZ", "G", "LOG", "PT", "S",
    "ERA001", "FIX", "PGH", "TRY201", "TRY400", "TRY401",
]
ignore = ["E111", "E114", "E117"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

**Exceptions:** Django projects add `DJ`. FastAPI projects add `FAST`.

**Source:** Ruff Docs

### Gates check only and do not rewrite [Recommended]

CI runs `ruff format --check` and `ruff check`. Do not auto-fix in CI. Use `--unsafe-fixes` only locally, and review the results one by one.

**Why:** Unsafe fixes may change runtime behavior.

**Source:** Ruff Docs

## Formatting

### Treat tool output as the formatting authority [Recommended]

Line length defaults to Ruff's 88. PEP 8 suggests 79, and a team convention may relax it to 99. Set one value per project. Do not fight the formatter by hand. Use `# fmt: off` only for data tables that truly need alignment, and apply it to whole statements.

**Source:** PEP 8, Ruff Docs

### Comments record only what the code cannot say [Required]

Do not write comments that restate the code, including docstrings that only repeat the name and type. API docs for published libraries and license headers are exempt. Express intent through naming and structure. Tool directives are exempt, for example `# type: ignore[...]`, `# noqa: ...`, `# fmt: off`, and `# nosec`. When the code cannot express a reason, write it in one line next to the code, such as an external system limit, counter-intuitive behavior, or a single-point trade-off like ignoring an error, a type assertion, or suppressing lint. Record caveats that span several places in the code in project docs. See [principles.md](../../general/principles.md).

**Tooling:** Use `ERA001` to check for commented-out code and `FIX` to check for to-do comments.

**Source:** Team

## Naming

### Name by object type [Recommended]

| Object | Rule | Example |
|---|---|---|
| Variables, functions, methods | `snake_case` | `order_total`, `fetch_user` |
| Classes | `CapWords`, acronyms in all caps | `OrderService`, `HTTPClient` |
| Exception classes | `CapWords`, ending in `Error` | `PaymentError` |
| Constants | `UPPER_SNAKE_CASE`, defined at module level | `MAX_RETRY_COUNT` |
| Modules | Short `snake_case`, no hyphens | `order_service.py` |
| Packages | Short lowercase, avoid underscores where possible | `billing` |
| Internal members | Single leading underscore | `_parse_row` |
| Booleans | Start with `is_`, `has_`, or `can_` | `is_active` |
| Type variables | Short `CapWords`, add `_co` for covariant and `_contra` for contravariant | `T`, `KT_contra` |

**Tooling:** `N` rule group.

**Source:** PEP 8, Google Python Style Guide

### Name the key in map variables [Required]

Name dict variables `<value>_by_<key>`. Do not repeat the type in the name, such as `user_dict`.

**Why:** Comments do not explain what the code is doing, so only the name can tell the reader what the key is.

**Example:**

Avoid:

```python
prices = {"A100": Decimal("9.90")}
```

Prefer:

```python
price_by_sku: dict[str, Decimal] = {"A100": Decimal("9.90")}
```

**Source:** Team

## Import order

### Sort imports in groups [Recommended]

Put imports at the top of the file, in three groups: standard library, third party, and this project, with a blank line between groups. Each `import` statement imports one module. Symbols from `typing` and `collections.abc` may be imported directly. Import container abstract types from `collections.abc`.

**Tooling:** `I` rule group, `UP035`.

**Source:** PEP 8, Google Python Style Guide

### Do not use wildcard imports [Recommended]

Do not write `from x import *`.

**Why:** The origin of names is unclear, names easily overwrite each other, and tools cannot check for undefined names.

**Tooling:** `F403`.

**Source:** PEP 8

### Prefer absolute imports [Recommended]

Write full paths for cross-package imports. Sibling modules in the same package may use single-dot relative imports. Do not use `..` to jump upward.

**Tooling:** `TID252`.

**Source:** PEP 8

## File organization

### Keep related code in one module [Recommended]

Python does not require one class per file. Keep closely related classes and functions together. Divide modules by responsibility, not by syntactic kind.

**Source:** Google Python Style Guide

### Put the entry point in a main function [Recommended]

Put the logic of executable files in `main()`, called from `if __name__ == "__main__":`. Importing a module produces no side effects.

**Why:** Otherwise, tests and tools run the program when they import the module. Multiprocessing `spawn` and `forkserver` also re-import the main module.

**Example:**

```python
def main() -> None:
    run_server()


if __name__ == "__main__":
    main()
```

**Source:** Google Python Style Guide, Python Standard Library Docs

### Avoid mutable global state [Recommended]

Do not put mutable business state at module level. Objects that live as long as the process, such as loggers, framework app and router objects, database engines, `ContextVar`, and type aliases, may stay at module level. When global state is truly needed, name it with a single leading underscore and access it through functions.

**Why:** Global state lets import order affect behavior, and tests contaminate each other.

**Tooling:** `PLW0603`.

**Source:** Google Python Style Guide
