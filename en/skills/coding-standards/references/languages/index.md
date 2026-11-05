# Language layer index

Determine the language from file extensions and project configuration, then read only the files relevant to the task in that language's directory.

| Language | Directory | Specialized files |
|---|---|---|
| TypeScript / JavaScript (including Node.js) | `typescript/` | [node.md](typescript/node.md) |
| Python | `python/` | [django.md](python/django.md), [fastapi.md](python/fastapi.md) |
| Go | `go/` | [web-services.md](go/web-services.md) |

Every language directory contains the following files:

| File | Read when |
|---|---|
| `style.md` | Tool configuration, formatting, naming, imports |
| `types.md` | Type design |
| `idioms.md` | Language idioms |
| `errors.md` | Error and exception handling |
| `concurrency.md` | Async and concurrency |
| `project.md` | Project structure, dependencies, and builds |
| `testing.md` | Writing tests |
| `security.md` | Language-specific security issues |

**Languages not covered:** Follow the language's official or community style guide and mainstream formatter, and tell the user this handbook has no detailed rules for that language. The domain layer and general layer still apply.
