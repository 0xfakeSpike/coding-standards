# Python project structure

This document sets rules for directories, package management, dependency locking, and builds. For general dependency principles, see [dependencies.md](../../general/dependencies.md).

## Directory structure

### Use the src layout [Recommended]

Put importable code in `src/<package>/` and tests in top-level `tests/`.

**Why:** Tests run against the installed package, so packaging omissions are caught early, and code in the current directory cannot accidentally shadow the installed package.

**Example:**

```text
project/
├── pyproject.toml
├── uv.lock
├── src/
│   └── billing/
│       ├── __init__.py
│       └── invoices.py
└── tests/
    └── test_invoices.py
```

**Exceptions:** Django projects use the directory containing `manage.py` as the root.

**Source:** Python Packaging Guide, pytest Docs

### Keep __init__.py light [Recommended]

`__init__.py` holds only necessary public exports. Do not perform I/O, open connections, or read configuration there.

**Why:** Importing the package runs this file, so side effects slow startup and easily cause circular imports.

### Declare command entry points in project.scripts [Recommended]

Declare command-line tools in `[project.scripts]`. Do not make users run source paths. This requires the project to define a build system.

**Source:** uv Docs

## Package management

### Use a virtual environment [Recommended]

Install dependencies only into the project's virtual environment, not the system Python. Do not commit `.venv` to version control.

**Source:** uv Docs

### Use uv for new projects, keep Poetry in existing Poetry projects [Recommended]

Use one package manager per project. Add and remove dependencies through commands. Do not edit lock files by hand.

| Operation | uv | Poetry |
|---|---|---|
| Add a dependency | `uv add httpx` | `poetry add httpx` |
| Add a dev dependency | `uv add --dev pytest` | `poetry add -G dev pytest` |
| Sync from the lock file | `uv sync --locked` | `poetry sync` |
| Run a command | `uv run pytest` | `poetry run pytest` |
| Validate the lock file | `uv lock --check` | `poetry check --lock` |

**Source:** uv Docs, Poetry Docs

### Declare dependencies in the standard tables [Recommended]

Put runtime dependencies in `[project] dependencies`, dev dependencies in `[dependency-groups]`, and optional features published to users in `[project.optional-dependencies]`.

**Why:** These tables all have standard definitions, so switching tools does not require rewriting them.

**Example:**

```toml
[project]
name = "billing"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["httpx>=0.27,<1"]

[dependency-groups]
dev = ["pytest>=9", "ruff>=0.17"]
```

**Source:** uv Docs, Poetry Docs

### Turn off packaging for dependency-only projects [Recommended]

For applications that do not need to build distribution packages, create them with `uv init --no-package` in uv, or set `package-mode = false` in Poetry.

**Source:** uv Docs, Poetry Docs

## Versions and locking

### Declare requires-python [Recommended]

State the minimum supported Python version in `pyproject.toml`. When `target-version` is not set, Ruff infers the target version from it.

**Source:** uv Docs, Ruff Docs

### Commit lock files for applications [Required]

Applications commit `uv.lock` or `poetry.lock`. A requirements file with full versions also works. CI and deployment install strictly from the lock file.

**Why:** Without a committed lock file, each build may get different versions, and problems cannot be reproduced.

**Tooling:** CI runs `uv sync --locked`, which fails immediately when the lock file is out of date.

**Source:** uv Docs, Poetry Docs

### Libraries declare ranges, applications lock versions [Recommended]

Libraries declare version ranges in `dependencies` and do not pin versions. A library's lock file affects only its own development and has no effect on consumers.

**Source:** Poetry Docs

### Upgrade dependencies explicitly [Recommended]

The lock file changes only on intentional upgrades. Upgrade a single package with `uv lock --upgrade-package <package>`.

**Source:** uv Docs

## Builds

### Build distribution packages with a standard backend [Recommended]

Declare the backend in `[build-system]`, and generate sdist and wheel with `uv build` or `poetry build`. Do not hand-write `setup.py`.

**Source:** uv Docs, Poetry Docs

### Install dependencies before the project in container images [Recommended]

First copy `pyproject.toml` and the lock file and install dependencies, then copy the source and install the project. This way, code changes do not rebuild the dependency layer. Do not install dev dependencies in production images.

**Example:**

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-install-project --no-dev
COPY . .
RUN uv sync --locked --no-dev --no-editable
```

**Source:** uv Docs

## Multiple modules or monorepos

### Use a uv workspace for tightly related packages [Recommended]

When several packages in one repository need to share a set of dependency versions, declare the members in `[tool.uv.workspace]`. All members share one lock file. Use `uv run --package <member>` to operate on a single member.

**Example:**

```toml
[tool.uv.workspace]
members = ["packages/*"]

[tool.uv.sources]
billing-core = { workspace = true }
```

**Source:** uv Docs

### Keep packages with conflicting dependencies out of one workspace [Recommended]

When members' dependency requirements conflict, or members need separate virtual environments, do not use a workspace. Create separate projects and have them reference each other with path dependencies. Poetry projects use path dependencies too.

**Source:** uv Docs
