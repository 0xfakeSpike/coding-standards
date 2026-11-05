# Python error handling

For general principles, see [error-handling.md](../../general/error-handling.md). This document covers only Python-specific mechanisms.

## Error mechanisms

### Use exceptions to express failure [Recommended]

On failure, raise an exception. Do not return `None`, `False`, or an error code and leave the caller to guess. For precondition violations, prefer built-in exceptions such as `ValueError` and `TypeError`. When a query may legitimately have no result, returning `T | None` is normal semantics, not an error. Express expected business failures with named exceptions. This is the Python form of "return an explicit result" in [error-handling.md](../../general/error-handling.md).

**Source:** Google Python Style Guide

### Catch specific exceptions [Required]

Do not write bare `except:`. It is equivalent to `except BaseException:` and catches `KeyboardInterrupt`, `SystemExit`, and `asyncio.CancelledError`. For system errors, catch `OSError` subclasses such as `FileNotFoundError`. Do not check `errno`. Use `except Exception:` only at isolation boundaries. See the "Framework boundaries" section.

**Tooling:** `E722`, `BLE001`.

**Source:** PEP 8, Google Python Style Guide

### Wrap only the statements that can fail in try [Recommended]

Put only the statements that raise the target exception in `try`. Put follow-up logic in `else`.

**Why:** The larger the block, the easier it is to mistake an exception of the same type from an unrelated statement for the target exception.

**Example:**

```python
try:
    value = collection[key]
except KeyError:
    return key_not_found(key)
else:
    return handle_value(value)
```

**Source:** PEP 8, Google Python Style Guide

## Custom errors and where they belong

### Inherit custom exceptions from Exception [Recommended]

Custom exceptions inherit from `Exception`, not `BaseException`. Names end in `Error` and do not repeat the module name. Divide types by the decision the caller has to make, not by where they are raised. Put information that programs need to read in attributes. Do not make callers parse message text.

**Tooling:** `N818`.

**Source:** PEP 8, Google Python Style Guide

### Define exceptions in the module that uses them [Required]

Define a specific exception in the module that uses it. Do not keep an error file shared by all modules.

**Source:** Team

### Base class per package, subclasses per module [Recommended]

Define one base class per package so callers can catch everything in one place.

**Example:**

```python
class BillingError(Exception):
    pass


class PaymentDeclinedError(BillingError):
    def __init__(self, order_id: int, reason: str) -> None:
        super().__init__(f"Payment declined order_id={order_id} reason={reason}")
        self.order_id = order_id
        self.reason = reason
```

**Source:** Google Python Style Guide

## Wrapping and inspection

### Preserve the root cause when converting exceptions [Required]

When raising a new exception in an `except` block, write `raise NewError(...) from err`. When you really need to hide the original exception, write `from None` and carry the necessary information into the new exception. To re-raise unchanged, write `raise`, not `raise err`. When the type stays the same and you only need to add information, call `err.add_note(...)` (Python 3.11+) and re-raise.

**Example:**

```python
try:
    charge_card(order)
except GatewayTimeoutError as err:
    raise PaymentError(f"Charge failed order_id={order.id}") from err
```

**Tooling:** `B904`, `TRY201`.

**Source:** PEP 8, Python Standard Library Docs

### Handle concurrent failures with except* [Recommended]

`asyncio.TaskGroup` merges subtask exceptions into an `ExceptionGroup`. Handle them by type with `except*`. A plain `except` cannot match exceptions inside the group. Python 3.11+.

**Example:**

```python
async def sync_all(shops: list[Shop]) -> None:
    try:
        async with asyncio.TaskGroup() as group:
            for shop in shops:
                group.create_task(sync_shop(shop))
    except* TimeoutError as timeouts:
        raise SyncTimeoutError(len(timeouts.exceptions)) from timeouts
```

**Source:** Python asyncio Docs, Python Standard Library Docs

### Use suppress for exceptions you can ignore [Recommended]

Once you have confirmed an exception can be ignored, write `contextlib.suppress(FileNotFoundError)`, not `except ...: pass`. Put a one-line comment with the reason next to it.

**Tooling:** `SIM105`, `S110`.

## Working with logging

### Use module loggers with lazy arguments [Recommended]

At the top of the module, write `logger = logging.getLogger(__name__)`. Do not call root logger functions such as `logging.info`. Write messages as `logger.info("user %s", user_id)`, not as f-strings.

**Why:** When the level is disabled, the message is not rendered, and the logging system can aggregate by the original template.

**Tooling:** `LOG015`, `G004`.

**Source:** Python logging Docs, Google Python Style Guide

### Log exceptions with logger.exception [Recommended]

In an `except` block, use `logger.exception("...")`, which adds the stack trace automatically. Do not also put the exception object in the message. Outside an `except` block, use `exc_info=err`.

**Tooling:** `TRY400`, `TRY401`, `LOG004`.

**Source:** Python logging Docs, Ruff Docs

### Configure logging once in the application [Recommended]

Libraries only create loggers and add no handlers other than `NullHandler`. The application calls `logging.config.dictConfig` once at the entry point and sets `disable_existing_loggers: False`. Otherwise, loggers created before configuration are disabled.

**Source:** Python logging Docs

## Framework boundaries

### Keep except Exception at isolation boundaries [Required]

Catch `Exception` only at boundaries such as request handlers, task loops, the outermost level of a thread, and tool execution entry points, and always log it or convert it into a result. Do not write it in the business layer.

**Tooling:** Write `# noqa: BLE001` at the boundary.

**Source:** Google Python Style Guide

### Convert errors at entry points and in the framework layer [Recommended]

A CLI `main()` catches expected errors, prints a message users can understand to stderr, and returns a nonzero exit code. Let the interpreter print the stack trace for program bugs. In web frameworks, exception handlers convert domain exceptions into responses, and the business layer does not import HTTP error types. See [django.md](django.md) and [fastapi.md](fastapi.md).

### Retrieve exceptions from background execution units [Required]

Exceptions from `Future` and `asyncio.Task` are raised only when you fetch the result. Call `.result()` or `await`, or use `TaskGroup`. Do not let failures disappear silently.

**Source:** Python asyncio Docs, Python Standard Library Docs
