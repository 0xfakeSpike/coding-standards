# Python 并发

本文规定线程、进程和 `asyncio` 的用法。Python 版本要求写在规则里。

## 并发模型

### 按负载选模型 【推荐】

| 负载 | 选择 |
|---|---|
| I/O 密集，有异步库 | `asyncio` |
| I/O 密集，只有同步库 | `ThreadPoolExecutor` 或 `asyncio.to_thread()` |
| CPU 密集 | `ProcessPoolExecutor` |
| CPU 密集，且依赖已验证支持自由线程 | 自由线程构建加线程池 |

**原因：** 默认构建中 GIL 只允许一个线程执行 Python 字节码，线程对 CPU 密集任务没有加速效果。

**来源：** Python 标准库文档、Python 自由线程说明

### 不默认依赖自由线程 【推荐】

自由线程构建需要 Python 3.13 及以上，默认构建仍带 GIL。采用前确认三件事：运行环境确实是该构建，所有 C 扩展都已声明支持，单线程开销可以接受。

**原因：** 导入未声明支持的扩展时，解释器会自动重新启用 GIL，只打印一条警告。

**来源：** Python 自由线程说明、PEP 703

### 多进程代码按 spawn 语义编写 【必须】

入口放在 `if __name__ == "__main__":` 下，传给子进程的函数和参数必须可 pickle。默认启动方式随平台和 Python 版本变化。

**原因：** 依赖 `fork` 继承全局状态的代码，换平台或升级 Python 后就会失败。

**来源：** Python 标准库文档

### async 函数中不调用阻塞操作 【必须】

`async def` 内不用 `time.sleep`、`requests`、同步文件和数据库调用。改用异步库，或用 `asyncio.to_thread()` 转到线程。CPU 密集的计算用 `run_in_executor` 配合进程池。网络类日志处理器同样会阻塞，用 `QueueHandler` 和 `QueueListener` 放到独立线程。

**原因：** 事件循环只有一个线程，一次阻塞会让所有任务一起停下。

**示例：**

```python
async def load_report(url: str) -> bytes:
    response = await asyncio.to_thread(requests.get, url, timeout=10)
    return response.content
```

**工具：** `ASYNC210`、`ASYNC221`、`ASYNC230`、`ASYNC251`。开发时用 `PYTHONASYNCIODEBUG=1` 发现慢回调。

**来源：** Python asyncio 文档、Python logging 文档、Ruff 文档

### 用 TaskGroup 管理并发任务 【必须】

并发子任务放在 `asyncio.TaskGroup` 中（Python 3.11+）。使用 `create_task()` 时必须保存引用，并在结束时等待。

**原因：** 事件循环只持有任务的弱引用，没有引用的任务可能在完成前被回收。`TaskGroup` 在一个任务失败时会取消其余任务，`gather` 不会。

**示例：**

反例：

```python
async def refresh(user_id: int) -> None:
    asyncio.create_task(rebuild_cache(user_id))
```

正例：

```python
async def refresh(user_ids: list[int]) -> None:
    async with asyncio.TaskGroup() as group:
        for user_id in user_ids:
            group.create_task(rebuild_cache(user_id))
```

**工具：** `RUF006`。

**来源：** Python asyncio 文档

## 取消与超时

### 不吞掉 CancelledError 【必须】

清理逻辑放在 `try/finally`。显式捕获 `asyncio.CancelledError` 后必须重新抛出。

**原因：** `TaskGroup` 和 `asyncio.timeout()` 都靠取消实现，吞掉后它们会行为异常。

**来源：** Python asyncio 文档

### 外部等待都设超时 【必须】

用 `async with asyncio.timeout(seconds):` 限定一段操作（Python 3.11+），异步函数本身不接收 `timeout` 参数。同步调用传 `timeout` 参数，比如 `requests.get(url, timeout=10)`、`Future.result(timeout=10)`。

**原因：** 没有超时的等待在对端无响应时会永远挂起。

**工具：** `S113`、`ASYNC109`。

**来源：** Python asyncio 文档、Requests 文档

## 共享状态

### 不依赖内置类型的原子性 【必须】

线程间共享的数据用 `queue.Queue` 传递，或者用 `threading.Lock` 保护，在自由线程构建下也一样。多个线程不共用同一个迭代器。

**原因：** 内置类型的内部锁只保护单次操作，不保护复合操作。

**来源：** Google Python Style Guide、Python 自由线程说明

### 请求级上下文用 ContextVar 【必须】

异步代码用 `contextvars.ContextVar` 保存上下文，不用 `threading.local`。

**原因：** 同一线程上的任务会共用线程局部变量，数据会串到其他任务。

**来源：** Python 标准库文档

### 区分线程锁和 asyncio 锁 【必须】

`asyncio.Lock` 等原语不是线程安全的，不能跨线程使用。在协程里也不要持有 `threading.Lock` 跨越 `await`。从其他线程操作事件循环时，用 `loop.call_soon_threadsafe()` 或 `asyncio.run_coroutine_threadsafe()`。

**来源：** Python asyncio 文档

## 资源限制

### 限制并发数 【必须】

**适用：** 线上服务

大批量任务用 `asyncio.Semaphore` 或固定 `max_workers` 的执行器限流。生产者和消费者之间的 `queue.Queue`、`asyncio.Queue` 都设 `maxsize`，让背压传回生产者。

**示例：**

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

**来源：** Python asyncio 文档

### 执行器用 with 管理 【推荐】

`ThreadPoolExecutor` 和 `ProcessPoolExecutor` 用 `with` 创建，退出时等待任务完成。提交的任务不等待同一执行器里的其他 `Future`，否则工作线程耗尽后会死锁。需要收尾的工作不用守护线程，它们会在退出时被直接中止。

**来源：** Python 标准库文档

## 常见竞态

### await 前后不假设状态不变 【必须】

协程在 `await` 处会让出执行权。检查和修改之间隔着 `await` 时，用 `asyncio.Lock` 保护，或者把这段操作交给数据库原子完成。

**示例：**

反例：

```python
async def reserve(stock: dict[str, int], sku: str) -> None:
    if stock[sku] > 0:
        await audit(sku)
        stock[sku] -= 1
```

正例：

```python
async def reserve(stock: dict[str, int], sku: str, lock: asyncio.Lock) -> None:
    async with lock:
        if stock[sku] > 0:
            await audit(sku)
            stock[sku] -= 1
```

