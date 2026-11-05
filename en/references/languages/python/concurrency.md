# Python concurrency

This document sets rules for threads, processes, and `asyncio`. Python version requirements are stated in the rules.

## Concurrency model

### Choose the model by workload [Recommended]

| Workload | Choice |
|---|---|
| I/O-bound, async libraries available | `asyncio` |
| I/O-bound, only sync libraries available | `ThreadPoolExecutor` or `asyncio.to_thread()` |
| CPU-bound | `ProcessPoolExecutor` |
| CPU-bound, and dependencies are verified to support free threading | Free-threaded build with a thread pool |

**Why:** In the default build, the GIL lets only one thread execute Python bytecode, so threads give no speedup for CPU-bound work.

**Source:** Python Standard Library Docs, Python Free-Threading Notes

### Do not rely on free threading by default [Recommended]

Free-threaded builds require Python 3.13 or later, and the default build still has the GIL. Before adopting one, confirm three things: the runtime really is that build, all C extensions declare support, and the single-thread overhead is acceptable.

**Why:** When an extension that does not declare support is imported, the interpreter automatically re-enables the GIL and only prints a warning.

**Source:** Python Free-Threading Notes, PEP 703

### Write multiprocessing code for spawn semantics [Required]

Put the entry point under `if __name__ == "__main__":`. Functions and arguments passed to child processes must be picklable. The default start method varies by platform and Python version.

**Why:** Code that relies on `fork` inheriting global state fails after a platform change or a Python upgrade.

**Source:** Python Standard Library Docs

### Do not call blocking operations in async functions [Required]

Inside `async def`, do not use `time.sleep`, `requests`, or synchronous file and database calls. Use async libraries instead, or move the call to a thread with `asyncio.to_thread()`. For CPU-bound computation, use `run_in_executor` with a process pool. Network log handlers block too: move them to a separate thread with `QueueHandler` and `QueueListener`.

**Why:** The event loop has only one thread. One blocking call stops all tasks.

**Example:**

```python
async def load_report(url: str) -> bytes:
    response = await asyncio.to_thread(requests.get, url, timeout=10)
    return response.content
```

**Tooling:** `ASYNC210`, `ASYNC221`, `ASYNC230`, `ASYNC251`. During development, use `PYTHONASYNCIODEBUG=1` to find slow callbacks.

**Source:** Python asyncio Docs, Python logging Docs, Ruff Docs

### Manage concurrent tasks with TaskGroup [Required]

Put concurrent subtasks in `asyncio.TaskGroup` (Python 3.11+). When you use `create_task()`, you must keep a reference and await the task at the end.

**Why:** The event loop holds only weak references to tasks, so a task with no reference may be garbage collected before it finishes. `TaskGroup` cancels the remaining tasks when one fails. `gather` does not.

**Example:**

Avoid:

```python
async def refresh(user_id: int) -> None:
    asyncio.create_task(rebuild_cache(user_id))
```

Prefer:

```python
async def refresh(user_ids: list[int]) -> None:
    async with asyncio.TaskGroup() as group:
        for user_id in user_ids:
            group.create_task(rebuild_cache(user_id))
```

**Tooling:** `RUF006`.

**Source:** Python asyncio Docs

## Cancellation and timeouts

### Do not swallow CancelledError [Required]

Put cleanup logic in `try/finally`. After explicitly catching `asyncio.CancelledError`, you must re-raise it.

**Why:** `TaskGroup` and `asyncio.timeout()` are both built on cancellation, and they misbehave when it is swallowed.

**Source:** Python asyncio Docs

### Set a timeout on every external wait [Required]

Use `async with asyncio.timeout(seconds):` to bound a block of operations (Python 3.11+). Async functions themselves do not take a `timeout` parameter. For sync calls, pass a `timeout` argument, for example `requests.get(url, timeout=10)` or `Future.result(timeout=10)`.

**Why:** A wait with no timeout hangs forever when the other end does not respond.

**Tooling:** `S113`, `ASYNC109`.

**Source:** Python asyncio Docs, Requests Docs

## Shared state

### Do not rely on the atomicity of built-in types [Required]

Pass data shared between threads through `queue.Queue`, or protect it with `threading.Lock`. This also applies to free-threaded builds. Do not share one iterator between threads.

**Why:** The internal locks of built-in types protect single operations only, not compound operations.

**Source:** Google Python Style Guide, Python Free-Threading Notes

### Use ContextVar for request-scoped context [Required]

In async code, store context in `contextvars.ContextVar`, not `threading.local`.

**Why:** Tasks on the same thread share thread-local variables, so data leaks into other tasks.

**Source:** Python Standard Library Docs

### Distinguish thread locks from asyncio locks [Required]

`asyncio.Lock` and similar primitives are not thread-safe and cannot be used across threads. In coroutines, do not hold a `threading.Lock` across an `await`. To operate on the event loop from another thread, use `loop.call_soon_threadsafe()` or `asyncio.run_coroutine_threadsafe()`.

**Source:** Python asyncio Docs

## Resource limits

### Limit concurrency [Required]

**Applies to:** production service

For large batches, limit concurrency with `asyncio.Semaphore` or an executor with a fixed `max_workers`. Set `maxsize` on every `queue.Queue` and `asyncio.Queue` between producers and consumers, so backpressure reaches the producer.

**Example:**

```python
async def fetch_all(urls: list[str], limit: int) -> None:
    semaphore = asyncio.Semaphore(limit)

    async def fetch_one(url: str) -> None:
        async with semaphore:
            await fetch(url)

    async with asyncio.TaskGroup() as group:
        for url in urls:
            group.create_task(fetch_one(url))
```

**Source:** Python asyncio Docs

### Use with to manage executors [Recommended]

Create `ThreadPoolExecutor` and `ProcessPoolExecutor` with `with`, which waits for tasks to finish on exit. Do not make a submitted task wait on another `Future` in the same executor, or it deadlocks once the worker threads run out. Do not use daemon threads for work that needs finishing: they are aborted outright at exit.

**Source:** Python Standard Library Docs

## Common races

### Do not assume state is unchanged across await [Required]

Coroutines yield control at `await`. When an `await` sits between a check and a change, protect them with `asyncio.Lock`, or have the database do the operation atomically.

**Example:**

Avoid:

```python
async def reserve(stock: dict[str, int], sku: str) -> None:
    if stock[sku] > 0:
        await audit(sku)
        stock[sku] -= 1
```

Prefer:

```python
async def reserve(stock: dict[str, int], sku: str, lock: asyncio.Lock) -> None:
    async with lock:
        if stock[sku] > 0:
            await audit(sku)
            stock[sku] -= 1
```
