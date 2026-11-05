# Python 错误处理

通用原则见 [error-handling.md](../../general/error-handling.md)。本文只讲 Python 的具体机制。

## 错误机制

### 用异常表达失败 【推荐】

失败时抛异常，不返回 `None`、`False` 或错误码让调用方猜测。违反前置条件时优先用 `ValueError`、`TypeError` 这类内置异常。查询确实可能没有结果时，返回 `T | None` 是正常语义，不算错误。预期内的业务失败用具名异常表达，这是 [error-handling.md](../../general/error-handling.md) 中“返回明确结果”在 Python 的写法。

**来源：** Google Python Style Guide

### 捕获具体异常 【必须】

不写裸 `except:`。它等同于 `except BaseException:`，会拦住 `KeyboardInterrupt`、`SystemExit` 和 `asyncio.CancelledError`。系统错误捕获 `FileNotFoundError` 这类 `OSError` 子类，不检查 `errno`。`except Exception:` 只用于隔离边界，见“框架边界”一节。

**工具：** `E722`、`BLE001`。

**来源：** PEP 8、Google Python Style Guide

### try 块只包住可能失败的语句 【推荐】

`try` 里只放会抛出目标异常的语句，后续逻辑放进 `else`。

**原因：** 块越大，越容易把无关语句抛出的同类异常当成目标异常处理。

**示例：**

```python
try:
    value = collection[key]
except KeyError:
    return key_not_found(key)
else:
    return handle_value(value)
```

**来源：** PEP 8、Google Python Style Guide

## 自定义错误及其归属

### 自定义异常继承 Exception 【推荐】

自定义异常继承 `Exception`，不继承 `BaseException`。名字以 `Error` 结尾，不重复模块名。类型按调用方要做的决定划分，不按抛出位置划分。需要程序读取的信息放在属性上，不让调用方解析消息文本。

**工具：** `N818`。

**来源：** PEP 8、Google Python Style Guide

### 异常定义在使用它的模块内 【必须】

具体异常定义在使用它的模块内，不设所有模块共用的错误文件。

**来源：** 团队

### 包有基类，模块有子类 【推荐】

包定义一个基类，方便调用方统一捕获。

**示例：**

```python
class BillingError(Exception):
    pass


class PaymentDeclinedError(BillingError):
    def __init__(self, order_id: int, reason: str) -> None:
        super().__init__(f"扣款被拒绝 order_id={order_id} reason={reason}")
        self.order_id = order_id
        self.reason = reason
```

**来源：** Google Python Style Guide

## 包装与判断

### 转换异常时保留根因 【必须】

在 `except` 块里抛新异常时写 `raise NewError(...) from err`。确实要隐藏原异常时写 `from None`，并把必要信息带进新异常。原样重抛写 `raise`，不写 `raise err`。类型不变、只需补充信息时，用 `err.add_note(...)`（Python 3.11+）后重抛。

**示例：**

```python
try:
    charge_card(order)
except GatewayTimeoutError as err:
    raise PaymentError(f"扣款失败 order_id={order.id}") from err
```

**工具：** `B904`、`TRY201`。

**来源：** PEP 8、Python 标准库文档

### 并发失败用 except* 处理 【推荐】

`asyncio.TaskGroup` 把子任务的异常合并成 `ExceptionGroup`。用 `except*` 按类型处理，普通 `except` 匹配不到组内的异常。Python 3.11+。

**示例：**

```python
async def sync_all(shops: list[Shop]) -> None:
    try:
        async with asyncio.TaskGroup() as group:
            for shop in shops:
                group.create_task(sync_shop(shop))
    except* TimeoutError as timeouts:
        raise SyncTimeoutError(len(timeouts.exceptions)) from timeouts
```

**来源：** Python asyncio 文档、Python 标准库文档

### 可以忽略的异常用 suppress 【推荐】

确认可以忽略时，写 `contextlib.suppress(FileNotFoundError)`，不写 `except ...: pass`。忽略的原因就近写一句注释。

**工具：** `SIM105`、`S110`。

## 日志配合

### 模块 logger 加惰性参数 【推荐】

模块顶部写 `logger = logging.getLogger(__name__)`，不调用 `logging.info` 这类根 logger 函数。消息写 `logger.info("user %s", user_id)`，不写 f-string。

**原因：** 级别被关闭时不用渲染消息，日志系统也能按原始模板聚合。

**工具：** `LOG015`、`G004`。

**来源：** Python logging 文档、Google Python Style Guide

### 记录异常用 logger.exception 【推荐】

在 `except` 块中用 `logger.exception("...")`，它会自动带上堆栈。不把异常对象再拼进消息。不在 `except` 块中时用 `exc_info=err`。

**工具：** `TRY400`、`TRY401`、`LOG004`。

**来源：** Python logging 文档、Ruff 文档

### 日志由应用统一配置 【推荐】

库只创建 logger，不添加 `NullHandler` 以外的处理器。应用在入口调用一次 `logging.config.dictConfig`，并设置 `disable_existing_loggers: False`，否则配置前创建的 logger 会被停用。

**来源：** Python logging 文档

## 框架边界

### except Exception 只放在隔离边界 【必须】

只有请求处理器、任务循环、线程最外层、工具执行入口这类边界才捕获 `Exception`，并且必须记录或转成结果。业务层不写。

**工具：** 在边界处写 `# noqa: BLE001`。

**来源：** Google Python Style Guide

### 入口和框架层转换错误 【推荐】

命令行的 `main()` 捕获预期错误，向 stderr 输出用户能理解的信息，并返回非零退出码，程序缺陷让解释器打印堆栈。Web 框架由异常处理器把领域异常转成响应，业务层不导入 HTTP 错误类型，见 [django.md](django.md) 和 [fastapi.md](fastapi.md)。

### 后台执行单元要取回异常 【必须】

`Future` 和 `asyncio.Task` 的异常只有在取结果时才抛出。调用 `.result()` 或 `await`，或者使用 `TaskGroup`，不让失败静默消失。

**来源：** Python asyncio 文档、Python 标准库文档

